---
title: "How to monitor OpenShift Virtualization VMs with Zabbix"
url: "https://developers.redhat.com/articles/2026/05/01/how-monitor-openshift-virtualization-vms-zabbix"
date: "Fri, 01 May 2026 07:01:31 +0000"
author: "Leonardo Araujo"
feed_url: "https://developers.redhat.com/blog/feed"
---
<p>In this article, I will demonstrate how to use Zabbix integrated with Prometheus/Thanos in the <a href="https://developers.redhat.com/products/openshift/virtualization">Red Hat OpenShift Virtualization</a> cluster. We will use low level discovery (LLD) to automate the discovery of all VMs, and thus monitor CPU, memory, network, etc. This is for users who need to create and monitor their OpenShift Virtualization using Zabbix, creating capacity alerts, applications, etc. I will not cover the installation of the Zabbix.</p><p><a></a></p><h2>Get started</h2><p>We will create a template using the <code>LLD</code> resource to process the collection of metrics for defining and creating the items and triggers. We will use Zabbix to connect to the Prometheus/Thanos API and have access to all metrics available in the environment, such as kubevirt metrics and infrastructure. Figure 1 shows the monitoring architecture.</p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/screenshot_2026-04-21_at_12.44.21_1.png"><img alt="A diagram of the monitoring architecture." height="313" src="https://developers.redhat.com/sites/default/files/styles/article_floated/public/screenshot_2026-04-21_at_12.44.21_1.png?itok=DzdECVGh" width="600" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 1: Monitoring architecture where Zabbix actively collects metrics from OpenShift Virtualization via Prometheus through a secure DMZ.</figcaption>
</figure>
<div class="markdown-heading"><p>Prerequisites:</p></div><p><a></a></p><ul><li>User with the cluster-admin cluster role</li><li>OpenShift Virtualization 4.16+</li><li>Zabbix server</li></ul><p>In this article, we will use the following versions:</p><ul><li>OpenShift Virtualization v4.18.23</li><li>Kubernetes v1.31.11</li><li>Zabbix 6.0.43</li></ul><p><a></a></p><div class="markdown-heading"><h2>Create a ServiceAccount</h2></div><p><a></a></p><p>Let's create a ServiceAccount in OpenShift Virtualization to use in our Zabbix connection to Prometheus/Thanos.</p><p>Using the <code>oc</code> CLI, connect to OpenShift as follows.</p><div class="highlight highlight-source-shell notranslate position-relative overflow-auto"><pre>$ oc project openshift-monitoring
$ oc create sa zabbix-sa
$ oc adm policy add-cluster-role-to-user cluster-monitoring-view -z zabbix-sa</pre><div class="zeroclipboard-container">Now let's create a token to perform our authentication.</div></div><div class="highlight highlight-source-shell notranslate position-relative overflow-auto"><pre>$ oc create token zabbix-sa -n openshift-monitoring --duration=8760h</pre><h3>Note</h3><span class="rhalert-content"><p>Using this command, we will have a token valid for 1 year. After this period, it is necessary to create a new one.</p></span></div><p>To create a token without expiration, create a secret of type <code>service-account-token</code>.</p><div class="highlight highlight-source-yaml notranslate position-relative overflow-auto"><pre><span class="pl-s">$ cat &lt;&lt;EOF | oc apply -f -</span>
<span class="pl-ent">apiVersion</span>: <span class="pl-c1">v1</span>
<span class="pl-ent">kind</span>: <span class="pl-s">Secret</span>
<span class="pl-ent">metadata</span>:
  <span class="pl-ent">name</span>: <span class="pl-s">zabbix-token                                </span><span class="pl-c"># &lt;----- Secret Name</span>
  <span class="pl-ent">namespace</span>: <span class="pl-s">openshift-monitoring</span>
  <span class="pl-ent">annotations</span>:
    <span class="pl-ent">kubernetes.io/service-account.name</span>: <span class="pl-s">zabbix-sa   </span><span class="pl-c"># &lt;----- ServiceAccount Name</span>
<span class="pl-ent">type</span>: <span class="pl-s">kubernetes.io/service-account-token           </span>
<span class="pl-s">EOF</span>
</pre><div class="zeroclipboard-container">To display the token, run the following command and save it for use in Zabbix.</div></div><div class="highlight highlight-source-shell notranslate position-relative overflow-auto"><pre>$ oc -n openshift-monitoring get secret zabbix-token -o jsonpath=<span class="pl-s pl-pds">'</span><span class="pl-s">{.data.token}</span><span class="pl-s pl-pds">'</span> <span class="pl-k">|</span> base64 -d</pre><div class="zeroclipboard-container">Collect the Thanos endpoint:</div></div><div class="highlight highlight-source-shell notranslate position-relative overflow-auto"><pre>$ oc -n openshift-monitoring get route thanos-querier -o jsonpath=<span class="pl-s pl-pds">'</span><span class="pl-s">{.spec.host}</span><span class="pl-s pl-pds">'</span></pre></div><p><a></a></p><div class="markdown-heading"><h2>Create a host group</h2></div><p><a></a></p><p>Let's create a host group to organize and create our OpenShift Virtualization hosts to monitor. In the left side menu, click on <code>Configuration</code> &gt; <code>Host groups</code> &gt; <code>Create host group</code> &gt; define the <code>Group name</code> and click <code>Add</code>.</p><p><a></a></p><p>Now we'll create a template to centralize all the items we want to monitor in OpenShift Virtualization so we can reuse it in other clusters.</p><p>In the left side menu, click on <code>Configuration</code> &gt; <code>Templates</code> &gt; <code>Create template</code> &gt; define the <code>Template name</code> &gt; in <code>Groups</code> enter the host group name created previously, then click <code>ADD</code>.</p><div class="markdown-heading"><h3>Create the host</h3></div><p><a></a></p><p>Now we will create a host, which will be the identifier for our OpenShift cluster. This will allow us to monitor more than one host (OpenShift cluster) with the same template.</p><p>In the left side menu, click on <code>Configuration</code> &gt; <code>Hosts</code> &gt; <code>Create host</code>, then define the following fields: <code>Host name</code>, <code>Templates</code>, <code>Groups</code>, <code>Description</code>. For templates and Groups, use the names created earlier.</p><p>Before saving, click on Macros, and let's add two variables (macros), which we will associate with our host (cluster) and use in our data collection.</p><p>Create the following Macros:</p><ul><li><code>{$PROM_URL}</code>: Add the Thanos URL that we collected in the last step of the ServiceAccount.</li><li><code>{$PROM_TOKEN}</code>: Add the token we created for our ServiceAccount.</li></ul><p>Now we can click <code>Add</code> and save our host (Figure 2).</p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content-full-width rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/zbx_host_macros_0.png"><img alt="Zabbix “Host macros” configuration screen showing PROM_URL with the Thanos/Prometheus endpoint and PROM_TOKEN used for API authentication." class="rhd-c-card__image" height="448" src="https://developers.redhat.com/sites/default/files/styles/article_full_width_1440px_w/public/zbx_host_macros_0.png?itok=5n7ebMgA" width="1440" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 2: Zabbix host macros configured with Prometheus API endpoint and authentication token.</figcaption>
</figure>
<div class="markdown-heading"><h3 class="heading-element">Low level discovery</h3></div><p><a></a></p><p><code>LLD</code> in Zabbix is a feature that automatically discovers resources within a monitored system and dynamically creates monitoring items, triggers, and graphs based on predefined rules.</p><p>Knowing the function of <code>LLD</code>, let's create two: one for virtual machines and another for virtualization nodes in OpenShift Virtualization.</p><p><a></a></p><p>We will create a <code>Discovery Rule (LLD)</code> that will discover all the virtual machines created in our OpenShift Virtualization cluster.</p><p>In the left side menu, click on <code>Configuration</code> &gt; <code>Templates</code> &gt; click on the template we created earlier &gt; click on <code>Discovery rules</code> in the top tab &gt; then click on <code>Create discovery rule</code>.</p><p>Fill in the following fields:</p><ul><li>Name: Add the name of the discovery that will be made.</li><li>Type: Select <code>HTTP agent</code></li><li>Key: Define a unique identifier for this execution.</li><li>URL: This will be our endpoint for querying metrics using the Macro <code>{$PROM_URL}/api/v1/query</code></li><li>Query fields: This field is responsible for passing our query promql to the endpoint in the URL field.<ul><li>Name: <code>query</code></li><li>Value: <code>kubevirt_vm_info</code></li></ul></li><li>Headers: Here we will add our token for authentication to the API using our token Macro.<ul><li>Name: <code>Authorization</code></li><li>Value: <code>Bearer {$PROM_TOKEN}</code></li></ul></li><li>Update interval: <code>1h</code> is the frequency at which our VM discovery will be run to discover new VMs.</li></ul><p>Our query will return a JSON output, but we need to filter the content we want in this output. We will use the <code>Preprocessing</code>feature.</p><p>Click on <code>Preprocessing</code> &gt; <code>Add</code> &gt; In Name, select <code>Javascript</code> &gt; click on <code>Parameters</code> and add the following script:</p><div class="highlight highlight-source-js notranslate position-relative overflow-auto"><pre><code class="language-javascript">var obj = JSON.parse(value);
var out = [];

for (var i = 0; i &lt; obj.data.result.length; i++) {
  var m = obj.data.result[i].metric;
  out.push({
    "{#VM}": m.name,
    "{#NAMESPACE}": m.namespace
  });
}

return JSON.stringify({ "data": out });</code></pre><h3>Note</h3><span class="rhalert-content"><p>This script will process all the output and filter only the following information: VM name and Namespace already creating Macros (variables) for later use.</p></span><p>To validate that our <code>Preprocessing</code> is working correctly, click <code>Test all steps</code>.</p></div><p>Check the <code>Get value from host</code> box and add the <code>Macros</code> values, adding the Thanos endpoint and the bearer token.&nbsp;</p><p>Then click <code>Get value and test</code> as shown in Figure 3.</p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content-full-width rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/zbx_lld_test_0.png"><img alt="The Zabbix test item window showing Prometheus API macros." class="rhd-c-card__image" height="922" src="https://developers.redhat.com/sites/default/files/styles/article_full_width_1440px_w/public/zbx_lld_test_0.png?itok=mpRhI1L_" width="1440" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 3: Testing Zabbix item with Prometheus API, validating macros and LLD preprocessing output.</figcaption>
</figure>
<p>Click <code>Add</code> to save our preprocessing and click <code>Add</code> again to save our discovery rule.</p><div class="markdown-alert markdown-alert-note"><h3>Note</h3><span class="rhalert-content"><p>With this test, we can see that our script is working correctly.</p></span><div class="markdown-heading"><h3>Create &nbsp;the item prototype</h3></div><p><a></a></p><p>With the <code>Item Prototype</code>, we will create specific queries such as CPU, memory, network, uptime, and phase using the discovery rule's Macros (variables), VM, and namespace.</p><p>Now click on <code>Item Prototype</code> within the discovery rule created earlier or <code>Configuration</code> &gt; <code>Templates</code> &gt; <code>Template Name</code> created earlier &gt; <code>Discovery Rule</code> created earlier &gt; <code>Item prototypes</code> &gt; <code>Create Item prototype</code>.</p><p>Fill in the following fields:</p><ul><li>Name: <code>VM {#VM} CPU usage</code></li><li>Type: Select <code>HTTP agent</code></li><li>Key: <code>kubevirt.vm.cpu[{#NAMESPACE},{#VM}]</code></li><li>Type of Information: <code>Numeric (float)</code></li><li>URL: <code>{$PROM_URL}/api/v1/query</code></li><li>Query fields: Promql to collect CPU usage from VMs<ul><li>Name: <code>query</code></li><li>Value: <code>sum by (name,namespace)(rate(kubevirt_vmi_cpu_usage_seconds_total{name="{#VM}",namespace="{#NAMESPACE}"}[5m]))</code></li></ul></li><li>Headers: Here we will add our token for authentication to the API using our token Macro.<ul><li>Name: <code>Authorization</code></li><li>Value: <code>Bearer {$PROM_TOKEN}</code></li></ul></li><li>Units: <code>cores</code></li><li>Update interval: <code>1m</code>, this is how often our item will be collected.</li></ul><p>Click on <code>Preprocessing</code> &gt; <code>Add</code> .&nbsp;</p><p>In Name, select <code>Javascript</code> and click on <code>Parameters</code> to add the following script:</p><div class="highlight highlight-source-js notranslate position-relative overflow-auto"><pre><code class="language-javascript">var obj = JSON.parse(value);

if (obj.data.result.length === 0) {
  return 0;
}

return obj.data.result[0].value[1];</code></pre><h3>Note</h3><span class="rhalert-content"><p>This script processes the collected information; if the VM has no CPU usage data, it will be displayed as 0.</p></span></div><p>Click <code>Add</code> to save our preprocessing and click <code>Add</code> again to save our item.</p><p>To speed up the creation of new items, let's clone this one by clicking on the created item. At the bottom of the page, click <code>Clone</code>.</p><p>In this Item, we will update the following fields:</p><ul><li>Name: <code>VM {#VM} CPU total</code></li><li>Key: <code>kubevirt.vm.cpu.request[{#NAMESPACE},{#VM}]</code></li><li>Type of Information: <code>Numeric (unsigned)</code></li><li>Query fields: Promql to collect CPU Total from VMs<ul><li>Name: <code>query</code></li><li>Value: <code>max by (namespace, name) (kubevirt_vm_resource_requests{resource="cpu", name="{#VM}", namespace="{#NAMESPACE}"} )</code></li></ul></li></ul><div class="markdown-alert markdown-alert-note"><h3>Note</h3><span class="rhalert-content"><p>The remaining fields should remain the same.</p></span><p class="markdown-alert-title">Click on <code>Preprocessing</code> &gt; <code>Add</code>. In Name, select <code>JSONPath</code>.</p><p class="markdown-alert-title">Click <code>Parameters</code> and add the value <code>$.data.result[0].value[1]</code>.</p></div><p>Click <code>Add</code> to save our preprocessing and click <code>Add</code> again to save our item.</p><p>Repeat the cloning process and add the following items.</p><p><strong>Memory usage:</strong></p><ul><li>Name: <code>VM {#VM} Memory Usage</code></li><li>Key: <code>kubevirt.vm.memory[{#NAMESPACE},{#VM}]</code></li><li>Type of Information: <code>Numeric (unsigned)</code></li><li>Query fields: Promql to collect Memory Usage from VMs<ul><li>Name: <code>query</code></li><li>Value: <code>sum by (name,namespace)(kubevirt_vmi_memory_used_bytes{name="{#VM}",namespace="{#NAMESPACE}"})</code></li></ul></li><li><p>In Preprocessing, select JavaScript and in parameter, add the following script:</p><div class="highlight highlight-source-js notranslate position-relative overflow-auto"><pre><code class="language-javascript">var obj = JSON.parse(value);

if (obj.data.result.length === 0) {
  return 0;
    }

return obj.data.result[0].value[1];</code></pre></div></li></ul><p><strong>Memory total:</strong></p><ul><li>Name: <code>VM {#VM} Memory total</code></li><li>Key: <code>kubevirt.vm.mem.request[{#NAMESPACE},{#VM}]</code></li><li>Type of Information: <code>Numeric (float)</code></li><li>Query fields: Promql to collect memory total from VMs<ul><li>Name: <code>query</code></li><li>Value: <code>max by (namespace, name)(kubevirt_vm_resource_requests{resource="memory", name="{#VM}", namespace="{#NAMESPACE}"})</code></li></ul></li><li>In preprocessing, select JSONPath and add &nbsp;<code>$.data.result[0].value[1]</code> in parameter.</li></ul><p>For this use case, we will monitor: CPU, memory, network, and VM status. You must create all the items you deem necessary to monitor, according to the needs of the environment.</p><div class="markdown-heading"><h3>Create the trigger prototype</h3></div><p><a></a></p><p>With the <code>Trigger Prototype</code>, we will create alerts based on the collected consumption items.</p><p>Now click on <code>Trigger Prototype</code> within the discovery rule created earlier or <code>Configuration</code> &gt; <code>Templates</code> &gt; <code>Template Name</code>created earlier &gt; <code>Discovery Rule</code> created earlier &gt; <code>Trigger prototypes</code> &gt; <code>Create trigger prototype</code></p><p>We will fill in the following fields:</p><ul><li>Name: <code>VM {#NAMESPACE}/{#VM} CPU usage high &gt;= 90%</code></li><li>Severity: <code>High</code></li><li><p>Expression:</p><div class="snippet-clipboard-content notranslate position-relative overflow-auto"><pre lang="promql"><code class="language-plaintext">(
avg(/Template OpenShift Virtualization/kubevirt.vm.cpu[{#NAMESPACE},{#VM}],5m)
/
last(/Template OpenShift Virtualization/kubevirt.vm.cpu.request[{#NAMESPACE},{#VM}])
) &gt; 0.90
and 
last(/Template OpenShift Virtualization/kubevirt.vm.status[{#VM}])=1
</code></pre><div class="zeroclipboard-container">&nbsp;</div></div></li><li>Description: <code>VM {#NAMESPACE}/{#VM} CPU utilization: {ITEM.LASTVALUE}</code></li></ul><p>Figure 4 illustrates the Zabbix trigger prototype.</p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/zbx_trigger_high_1.png"><img alt="Zabbix trigger prototype configuration screen showing a high CPU usage alert." height="466" src="https://developers.redhat.com/sites/default/files/styles/article_floated/public/zbx_trigger_high_1.png?itok=b15H7LSQ" width="600" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 4: Zabbix trigger prototype configured to alert on high VM CPU usage using LLD macros and Prometheus-based metrics.</figcaption>
</figure>
<div class="markdown-alert markdown-alert-note"><h3>Note</h3><span class="rhalert-content"><p>Repeat the cloning process, change the value from 0.90 to 0.70 for example, and create the alert with a Warning serverity level.</p><p>In this expression, we are dividing the total by the consumption, and the result needs to be greater than 0.90 (90%), ensuring that we only do this for the VMs that are running.</p></span><p>Create alerts with different severities, according to the items that were discovered (Figure 5).</p></div><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/zbx_triggers_0.png"><img alt="The Zabbix interface displays a list of trigger prototypes for virtual machines." height="302" src="https://developers.redhat.com/sites/default/files/styles/article_floated/public/zbx_triggers_0.png?itok=06x9HbB3" width="600" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 5: The Zabbix trigger prototypes list shows multiple alerts for VM CPU, memory, and status based on LLD discovery.</figcaption>
</figure>
<div class="markdown-heading"><h3>Create the graph prototype</h3></div><p><a></a></p><p>With <code>Graph Prototype</code>, we can create specific graphs, such as a network graph.</p><p>Click on <code>Graph Prototype</code> within the discovery rule created earlier or <code>Configuration</code> &gt; <code>Templates</code> &gt; <code>Template Name</code> created earlier &gt; <code>Discovery Rule</code> created earlier &gt; <code>Graph prototypes</code> &gt; <code>Create graph prototype</code>.</p><p>Fill in the following fields:</p><ul><li>Name: <code>VM {#NAMESPACE}/{#VM} Network throughput</code></li><li>Items:<ul><li>Add prototype:<ul><li>Select the prototype items created for network RX and TX.</li></ul></li></ul></li></ul><p>After adding the items, change the <code>Function</code> field to <code>avg</code>, adjust the item colors as desired, and click <code>Add</code>.</p><div class="markdown-heading"><h4 class="heading-element focus-visible">Create discovery rule (LLD) for VMs</h4></div><p><a></a></p><p>Now, to make our monitoring more complete, let's create an <code>LLD</code> of nodes for virtualization. That is, nodes where the VMs run.</p><p>In the left side menu, click on <code>Configuration</code> &gt; <code>Templates</code>&nbsp;</p><p>Click on the template we created earlier.</p><p>Click on <code>Discovery rules</code> in the top tab.&nbsp;</p><p>Then click on <code>Create discovery rule</code>.</p><p>Fill in the following fields as follows:</p><ul><li>Name: OpenShift - Node discovery</li><li>Type: Select <code>HTTP agent</code></li><li>Key: openshift.node.discovery</li><li>URL: <code>{$PROM_URL}/api/v1/query</code></li><li>Query fields: This field is responsible for passing our query promql to the endpoint in the URL field.<ul><li>Name: <code>query</code></li><li>Value: <code>count by (node) (kube_node_labels{label_kubevirt_io_schedulable="true"})</code></li></ul></li><li>Headers: Here we will add our token for authentication to the API using our Token Macro.<ul><li>Name: <code>Authorization</code></li><li>Value: <code>Bearer {$PROM_TOKEN}</code></li></ul></li><li>Update interval: <code>1h</code>, this is the frequency at which our VM discovery will be run, to discover new VMs.</li></ul><p>Click on <code>Preprocessing</code> &gt; <code>Add</code>.</p><p>In Name, select <code>Javascript</code>.&nbsp;</p><p>Click on <code>Parameters</code> and add the following script:</p><div class="highlight highlight-source-js notranslate position-relative overflow-auto"><pre><code class="language-javascript">var obj = JSON.parse(value);
var out = [];

for (var i = 0; i &lt; obj.data.result.length; i++) {
  out.push({ "{#NODE}": obj.data.result[i].metric.node });
}

return JSON.stringify({ "data": out });</code></pre><div class="zeroclipboard-container">&nbsp;</div><h3>Note</h3><span class="rhalert-content"><p>This script will process all the output and filter only the following information: NODE Name, already creating Macros (variables) for later use.</p></span></div><p>To validate that our <code>Preprocessing</code> is working correctly, click <code>Test all steps</code> (Figure 6).</p><p>Check the <code>Get value from host</code> box and add the <code>Macros</code> values, adding the Thanos endpoint and the bearer token. Then click <code>Get value and test</code>.</p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content-full-width rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/zbx_lld_nodes_test_0.png"><img alt="The Zabbix test item window displays the Prometheus API response for node discovery." class="rhd-c-card__image" height="915" src="https://developers.redhat.com/sites/default/files/styles/article_full_width_1440px_w/public/zbx_lld_nodes_test_0.png?itok=_mMC7rqn" width="1440" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 6: Testing Zabbix LLD for node discovery using Prometheus API and JavaScript preprocessing.</figcaption>
</figure>
<p>Click <code>Add</code> to save our preprocessing and click <code>Add</code> again to save the item.</p><div class="markdown-heading"><h4 class="heading-element">Create the item prototype</h4></div><p><a></a>With the <code>LLD</code> of nodes, we can also create <code>ITEM Prototype</code> such as CPU, memory, network, uptime, and phase using the discovery rule's Macros (variables), <code>NODE</code>.</p><p>Now click on <code>Item Prototype</code> within the discovery rule created earlier or <code>Configuration</code> &gt; <code>Templates</code> &gt; <code>Template Name</code> created earlier &gt; <code>Discovery Rule</code> created earlier &gt; <code>Item prototypes</code> &gt; <code>Create Item prototype</code>.</p><p>Fill in the following fields:</p><ul><li>Name: <code>Node {#NODE} Ready status</code></li><li>Type: Select <code>HTTP agent</code></li><li>Key: <code>node.ready[{#NODE}]</code></li><li>Type of Information: <code>Numeric (unsigned)</code></li><li>URL: <code>{$PROM_URL}/api/v1/query</code></li><li>Query fields: Promql to collect CPU usage from VMs<ul><li>Name: <code>query</code></li><li>Value: <code>max by (node) (kube_node_status_condition{ condition="Ready", status="true", node="{#NODE}" })</code></li></ul></li><li>Headers: Here we will add our token for authentication to the API using our token macro.<ul><li>Name: <code>Authorization</code></li><li>Value: <code>Bearer {$PROM_TOKEN}</code></li></ul></li><li>Update interval: <code>30s</code>, this is how often our item will be collected.</li></ul><p>Click on <code>Preprocessing</code> &gt; <code>Add</code>&nbsp;</p><p>In Name, select <code>Javascript</code>.</p><p>Click on <code>Parameters</code> and add the following script:</p><div class="highlight highlight-source-js notranslate position-relative overflow-auto"><pre><code class="language-javascript">var obj = JSON.parse(value);

if (obj.data.result.length === 0) return 0;

return obj.data.result[0].value[1];</code></pre><div class="zeroclipboard-container">&nbsp;</div></div><div class="markdown-alert markdown-alert-note"><h3>Note</h3><span class="rhalert-content"><p>This JavaScript will simply ensure that we get 0 as a return value if it's null or if the value is different from what's expected.</p></span><p>Click <code>Add</code> to save our preprocessing and click <code>Add</code> again to save the item. Repeat the previous steps to create new items, triggers, and graph prototypes.</p></div><p>In this <code>LLD</code>, we create items and triggers for the CPU, memory, network, boot time and ready status. Download the XML template if needed by clicking this <a href="https://github.com/leoaaraujo/articles/blob/master/openshift-virt-zabbix/files/zbx_export_templates.xml">link</a>.</p><div class="markdown-alert markdown-alert-important"><h3>Important</h3><span class="rhalert-content"><p>This is not a complete template. You should use it as a base and customize according to the environment/cluster.</p></span></div><div class="markdown-heading"><h3 class="heading-element">Viewing the collected items</h3></div><p><a></a></p><p>Now that we have created our host cluster and template with the LLDs and their items, let's check if the data was collected correctly by going to <code>Monitoring</code> &gt; <code>Latest Data</code>.</p><p>Figure 7 shows the items collected by LLD <code>KubeVirt - VM discovery</code>.</p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content-full-width rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/zbx_latest_vm_0.png"><img alt="The Zabbix monitoring interface shows the latest data for multiple VMs." class="rhd-c-card__image" height="830" src="https://developers.redhat.com/sites/default/files/styles/article_full_width_1440px_w/public/zbx_latest_vm_0.png?itok=brrSkqGj" width="1440" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 7: The Zabbix latest data view displays collected metrics for discovered virtual machines.</figcaption>
</figure>
<p>Figure 8 shows the items collected by LLD <code>OpenShift - Node discovery</code>.</p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content-full-width rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/zbx_latest_nodes_0.png"><img alt="The Zabbix monitoring interface shows the latest data for multiple OpenShift worker nodes." class="rhd-c-card__image" height="825" src="https://developers.redhat.com/sites/default/files/styles/article_full_width_1440px_w/public/zbx_latest_nodes_0.png?itok=rKXo7Tx_" width="1440" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 8: The Zabbix latest data view displays collected metrics for discovered OpenShift nodes.</figcaption>
</figure>
<div class="markdown-alert markdown-alert-note"><h3>Note</h3><span class="rhalert-content"><p>It's important to note that the "Without data" filter is set to 0, meaning all items are being collected correctly.</p></span></div><p><a></a></p><p>With all our items collected correctly, we can view our dashboard and identify if there are any active alerts (Figure 9).</p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/zbx_dashboard_0.png"><img alt="The Zabbix global dashboard displays system information, problem alerts, and severity levels for VMs and OpenShift nodes." height="328" src="https://developers.redhat.com/sites/default/files/styles/article_floated/public/zbx_dashboard_0.png?itok=wdoCbHw5" width="600" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 9: The Zabbix dashboard shows a global view of system health, including alerts for virtual machines and OpenShift nodes.</figcaption>
</figure>
<div class="markdown-heading"><h2 class="heading-element">Wrap up</h2></div><p><a></a></p><p>Monitoring OpenShift Virtualization with Zabbix demonstrates how easily external tools can integrate with the platform's native observability stack. By using Prometheus and Thanos, OpenShift Virtualization exposes comprehensive metrics that can be consumed by Zabbix through simple HTTP queries.</p><p>This enables effective monitoring of virtual machines and infrastructure without additional agents, using features such as LLD for dynamic discovery and automation. Ultimately, this approach combines the flexibility of Zabbix with the power of OpenShift Virtualization's integrated monitoring, providing a scalable and efficient solution for modern environments.</p><p><a></a></p><p>For more details and other configurations, start with these reference documents:</p><ul><li><a href="https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/virtualization/monitoring">OpenShift Virtualization - Monitoring</a></li><li><a href="https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.18/html-single/accessing_metrics/index">OpenShift 4.18 - Accessing metrics</a></li><li><a href="https://kubevirt.io/user-guide/user_workloads/component_monitoring/">KubeVirt.io - Monitoring</a></li><li><a href="https://www.zabbix.com/documentation/6.0/en/manual/discovery/low_level_discovery">Zabbix 6.0 - Low Level Discovery</a></li><li><a href="https://www.zabbix.com/documentation/6.0/en/manual/config/items/itemtypes/http">Zabbix 6.0 - Item HTTP Agent</a></li><li><a href="https://www.zabbix.com/documentation/6.0/en/manual/xml_export_import/templates">Zabbix 6.0 - Export and Import Templates</a></li></ul></div>

The post <a href="https://developers.redhat.com/articles/2026/05/01/how-monitor-openshift-virtualization-vms-zabbix" title="How to monitor OpenShift Virtualization VMs with Zabbix">How to monitor OpenShift Virtualization VMs with Zabbix</a> appeared first on <a href="https://developers.redhat.com/blog" title="Red Hat Developer">Red Hat Developer</a>.
<br /><br />

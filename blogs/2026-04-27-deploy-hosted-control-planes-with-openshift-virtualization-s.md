---
title: "Deploy hosted control planes with OpenShift Virtualization: Split hub"
url: "https://developers.redhat.com/articles/2026/04/27/deploy-hosted-control-planes-openshift-virtualization-split-hub"
date: "Mon, 27 Apr 2026 07:01:04 +0000"
author: "Prakash Rajendran"
feed_url: "https://developers.redhat.com/blog/feed"
---
<p dir="ltr"><span>In&nbsp;</span><a href="https://developers.redhat.com/articles/2026/04/20/deploy-hosted-control-planes-openshift-virtualization">Part 1</a><span> of this article series, we deployed&nbsp;</span><a href="https://redhat.com/en/technologies/management/advanced-cluster-management">Red Hat Advanced Cluster Management for Kubernetes</a><span>, hosted control plane (HCP) and&nbsp;</span><a href="https://developers.redhat.com/products/openshift/virtualization">Red Hat OpenShift Virtualization</a><span> entirely inside one&nbsp;</span><a href="https://developers.redhat.com/products/openshift/overview">Red Hat OpenShift</a><span> cluster. While the all-in-one model is simple and easy to understand, enterprises rarely operate this way in production. Large organizations prefer to separate fleet management from cluster hosting, allowing different teams and infrastructure zones to scale independently and cleanly.</span></p><h2><span>OpenShift clusters</span></h2><p dir="ltr"><span>In this topology, we need two OpenShift clusters.</span></p><p dir="ltr"><span>Cluster A (the Red Hat Advanced Cluster Management hub):</span></p><ul><li dir="ltr"><span>Runs Red Hat Advanced Cluster Management</span></li><li dir="ltr"><span>Does not host control planes</span></li><li dir="ltr"><span>Does not run worker VMs</span></li><li dir="ltr"><span>Manages lifecycle and governance of many clusters</span></li></ul><p dir="ltr"><span>Cluster B (management/hosting cluster </span><a href="https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.5/html-single/multicluster_engine/index">multicluster engine for Kubernetes</a><span>, hosted control plane, and OpenShift Virtualization):</span></p><ul><li dir="ltr"><span>Runs multicluster engine for Kubernetes</span></li><li dir="ltr"><span>Runs hosted control plane</span></li><li dir="ltr"><span>Runs OpenShift Virtualization</span></li><li dir="ltr"><span>Creates HostedClusters and NodePools</span></li><li dir="ltr"><span>Hosts both control plane pods and worker VMs</span></li></ul><p dir="ltr"><span>Hosted clusters created in Cluster B are then imported into Cluster A for Day-2 operations (Figure 1).</span></p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/image1_255.png"><img alt="A diagram shows the hub and spoke architecture." height="448" src="https://developers.redhat.com/sites/default/files/styles/article_floated/public/image1_255.png?itok=cfL4RkfU" width="600" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 1: Hub and spoke architecture.</figcaption>
</figure>
<p dir="ltr"><span>Enterprises choose this model for these reasons:</span></p><ul><li dir="ltr"><span>Clear separation of duties</span><ul><li dir="ltr"><span>Hub team controls governance, policy, visibility</span></li><li dir="ltr"><span>Hosting platform team manages actual cluster creation + compute resources</span></li></ul></li><li dir="ltr"><span>Red Hat Advanced Cluster Management cluster stays lightweight</span><ul><li dir="ltr"><span>No workload hosting or control-plane workloads overload the hub.</span></li></ul></li><li dir="ltr"><span>Multicluster engine for Kubernetes cluster can scale independently</span><ul><li dir="ltr"><span>Add storage, CPU, worker nodes as demand for HostedClusters grows.</span></li></ul></li><li dir="ltr"><span>Supports multi-datacenter strategies</span><ul><li dir="ltr"><span>Hub cluster can run centrally; hosting cluster may run in different regions.</span></li></ul></li></ul><p dir="ltr"><span>Common prerequisites:</span></p><ul><li><p dir="ltr"><span>DNS entries for the hub cluster pointing to the IP from the same subnet as the nodes of the cluster.</span></p></li><li><p dir="ltr"><span>API:&nbsp;</span><a href="http://api.cluster.example.com">api.cluster.example.com</a></p></li><li><p dir="ltr"><span>Ingress: *.apps.cluster.example.com</span></p></li><li><p dir="ltr"><span>Plan for firewall, LB, DNS entries across clusters</span></p></li></ul><p dir="ltr"><span>Tools/credentials required:</span></p><ul><li dir="ltr"><span>OpenShift Installer</span></li><li dir="ltr"><span>OC CLI</span></li><li dir="ltr"><span>Pull Secret</span></li><li dir="ltr"><span>SSH keys</span></li><li dir="ltr"><code><span>hcp</span></code><span> CLI</span></li><li dir="ltr"><code><span>clusteradm</span></code><span> CLI plug-in</span></li></ul><h2>Cluster A (Red Hat Advanced Cluster Management)</h2><p dir="ltr"><span>For demonstration purposes, we tested this on VMs running on vSphere using nested virtualization.&nbsp;</span></p><p dir="ltr">Nodes Sizing (Compact 3 node cluster):<strong> </strong><span>3 x Master/Worker Nodes - 8 vCPU / 32G Memory / 1 x 125 GB disk</span></p><p dir="ltr"><span>Once all the pre-requisite are met, install the OpenShift cluster using your preferred installation method.</span></p><p dir="ltr">Then install the Red Hat Advanced Cluster Management operator .</p><p dir="ltr"><span>Check whether all the required operators are installed on Cluster A (Figure 2).</span></p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/image3_142.png"><img alt="This table shows the operators installed on Cluster A." height="75" src="https://developers.redhat.com/sites/default/files/styles/article_floated/public/image3_142.png?itok=ZkJ_o_BD" width="600" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 2: Installed Operators on Cluster A</figcaption>
</figure>
<p dir="ltr"><strong>Prerequisites for Cluster B (multicluster engine for Kubernetes + hosted control plane + OCP-V):</strong></p><p dir="ltr"><span>For the demonstration purpose, this was tested on VMs running on vSphere using nested virtualization.</span></p><p dir="ltr"><span>3 x Master Nodes - 8 vCPU / 32G Memory / 1 x 125 GB disk</span></p><p dir="ltr"><span>3 x Worker Nodes - 16 vCPU / 48G Memory / 1 x 125 GB disk + 1 x 500 GB disk</span></p><p dir="ltr"><span>To keep things simple, we are going to use LVM Storage Class for both&nbsp;etcd PVs of hosted clusters and for OCP-V requirements. But in the real world scenario, consult official Red Hat documentation for choosing the right storage classes.</span></p><p dir="ltr">Install LVM storage operator.&nbsp;</p><p dir="ltr"><span>Once the LVMS operator is installed, create the following CR.</span></p><pre><code class="language-plaintext">$ cat &lt;&lt;EOF | oc apply -f -
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
&nbsp;&nbsp;name: lvmcluster-sample
&nbsp;&nbsp;namespace: openshift-storage
spec:
&nbsp;&nbsp;storage:
&nbsp;&nbsp;&nbsp;&nbsp;deviceClasses:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- fstype: xfs
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;thinPoolConfig:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;chunkSizeCalculationPolicy: Static
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;metadataSizeCalculationPolicy: Host
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;sizePercent: 90
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;name: thin-pool-1
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;overprovisionRatio: 10
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;default: true
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;name: vg1
EOF</code></pre><p dir="ltr"><span>Once the LVMCluster is available, you should see a new StorageClass created by name </span><code><span>lvms-vg1</span></code><span> but we need to create a new StorageClass with&nbsp;</span><code><span>VolumeBindingMode: Immediate</span></code><span>.</span></p><p dir="ltr"><strong>Note</strong><span>: This step is required only when you are using LVM StorageClass for OpenShift Virtualization and for testing purposes. In the production scenario, you might be choosing a different storage solution which supports RWX and&nbsp;</span><code><span>VolumeBindingMode: WaitForFistConsumer</span></code><span>&nbsp;is recommended.</span></p><pre><code class="language-plaintext">$ cat &lt;&lt;EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
&nbsp;&nbsp;name: lvm-immediate
&nbsp;&nbsp;annotations:
&nbsp;&nbsp;&nbsp;&nbsp;description: Provides RWO and RWOP Filesystem &amp; Block volumes
&nbsp;&nbsp;&nbsp;&nbsp;storageclass.kubernetes.io/is-default-class: "true"
&nbsp;&nbsp;labels:
&nbsp;&nbsp;&nbsp;&nbsp;owned-by.topolvm.io/group: lvm.topolvm.io
&nbsp;&nbsp;&nbsp;&nbsp;owned-by.topolvm.io/kind: LVMCluster
&nbsp;&nbsp;&nbsp;&nbsp;owned-by.topolvm.io/name: lvmcluster-sample
&nbsp;&nbsp;&nbsp;&nbsp;owned-by.topolvm.io/namespace: openshift-storage
&nbsp;&nbsp;&nbsp;&nbsp;owned-by.topolvm.io/version: v1alpha1
provisioner: topolvm.io
parameters:
&nbsp;&nbsp;csi.storage.k8s.io/fstype: xfs
&nbsp;&nbsp;topolvm.io/device-class: vg1
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
EOF</code></pre><p dir="ltr"><span>Make sure to remove the default SC annotation from </span><code><span>lvms-vg1</span></code><span>.</span></p><pre><code class="language-plaintext">$ oc patch storageclass lvms-vg1 \
&nbsp;&nbsp;-p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": null}}}'</code></pre><p dir="ltr">Install MetalLB operator.</p><p dir="ltr"><span>Configure IPPool &amp; L2Advertisement as per the </span><a href="https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/hosted_control_planes/deploying-hosted-control-planes#hcp-metallb_hcp-deploy-virt"><span>official documentation</span></a><span>.</span></p><p dir="ltr"><span>Once you've installed the MetalLB operator, create the following:</span></p><pre><code class="language-plaintext">$ cat &lt;&lt;EOF | oc apply -f -
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
&nbsp;&nbsp;name: metallb
&nbsp;&nbsp;namespace: metallb-system
EOF

$ cat &lt;&lt;EOF | oc apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
&nbsp;&nbsp;name: metallb
&nbsp;&nbsp;namespace: metallb-system
spec:
&nbsp;&nbsp;addresses:
&nbsp;&nbsp;- 192.168.34.205-192.168.34.215
EOF

$ cat &lt;&lt;EOF | oc apply -f -
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
&nbsp;&nbsp;name: l2advertisement
&nbsp;&nbsp;namespace: metallb-system
spec:
&nbsp;&nbsp;ipAddressPools:
&nbsp;&nbsp;&nbsp;- metallb
EOF</code></pre><p dir="ltr">Patch network operator:</p><pre><code class="language-plaintext">$ oc patch ingresscontroller -n openshift-ingress-operator default \
&nbsp;&nbsp;--type=json \
&nbsp;&nbsp;-p '[{ "op": "add", "path": "/spec/routeAdmission", "value": {wildcardPolicy: "WildcardsAllowed"}}]'</code></pre><p dir="ltr">Install OpenShift Virtualization operator and hyperconverged CR.&nbsp;</p><p dir="ltr"><span>Once you've installed all the operators successfully, it’s time to test by creating a simple VM to make sure OCP Virtualization is functioning. Once VM validation is successful, proceed with next steps to configure a multicluster engine for Kubernetes.</span></p><p dir="ltr">Install multicluster engine for the Kubernetes operator.</p><p dir="ltr"><span>Once you've installed the multicluster engine for Kubernetes successfully, make sure the hub cluster is seen as the managed cluster.</span></p><pre><code class="language-plaintext">$ oc get managedclusters local-cluster</code></pre><p dir="ltr"><span>Check whether all the required operators are installed (Figure 3).</span></p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/image6_67.png"><img alt="This table shows the operators installed on Cluster B." height="204" src="https://developers.redhat.com/sites/default/files/styles/article_floated/public/image6_67.png?itok=XATAIdCb" width="600" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 3: Installed operators on Cluster B.</figcaption>
</figure>
<p dir="ltr"><span>Prepare the multicluster engine for Kubernetes cluster before importing it on Red Hat Advanced Cluster Management cluster.</span></p><p dir="ltr"><span>It is important to complete all the steps outlined in this </span><a href="https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/multicluster_engine_operator_with_red_hat_advanced_cluster_management/mce-acm-integration"><span>official documentation</span></a><span> to manage the multicluster engine for Kubernetes cluster from Red Hat Advanced Cluster Management and auto-discover hosted clusters imported using the policies.</span></p><p dir="ltr"><span>Do not proceed with the next section before completing this section. Once the multicluster engine for the Kubernetes cluster is imported, we can see Red Hat Advanced Cluster Management (Cluster A) in Figure 4.</span></p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/image5_83.png"><img alt="This table shows the cluster view of Cluster A." height="169" src="https://developers.redhat.com/sites/default/files/styles/article_floated/public/image5_83.png?itok=lOpQk7iK" width="600" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 4: Cluster view from Cluster A.</figcaption>
</figure>
<p dir="ltr"><span>On the multicluster engine for Kubernetes cluster, there is just a local cluster (Figure 5).</span></p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/image2_168.png"><img alt="This table shows the cluster view of Cluster B." height="155" src="https://developers.redhat.com/sites/default/files/styles/article_floated/public/image2_168.png?itok=Rowi4Gof" width="600" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 5: Cluster view from Cluster B.</figcaption>
</figure>
<h2>Create a hosted cluster on Cluster B</h2><p dir="ltr"><span>You cannot create a hosted cluster from the Red Hat Advanced Cluster Management hub cluster. Connect to the multicluster engine for Kubernetes cluster and run&nbsp;hcp create cluster&nbsp;command to create the hosted cluster.&nbsp;</span></p><p dir="ltr"><span>This single&nbsp;hcp create cluster command provisions the hosted cluster on the multicluster engine for Kubernetes (Cluster B) using nodepools VMs using KubeVirt.</span></p><pre><code class="language-plaintext">$ export KUBECONFIG=kubeconfig-mce

$ hcp create cluster kubevirt \
&nbsp;&nbsp;&nbsp;&nbsp;--name mce-hc1 \
&nbsp;&nbsp;&nbsp;&nbsp;--pull-secret&nbsp;pull-secret.txt&nbsp;\
&nbsp;&nbsp;&nbsp;&nbsp;--node-pool-replicas 2 \
&nbsp;&nbsp;&nbsp;&nbsp;--memory 8Gi \
&nbsp;&nbsp;&nbsp;&nbsp;--cores 2 \
&nbsp;&nbsp;&nbsp;&nbsp;--etcd-storage-class=lvm-immediate \
&nbsp;&nbsp;&nbsp;&nbsp;--namespace clusters \
&nbsp;&nbsp;&nbsp;&nbsp;--release-image quay.io/openshift-release-dev/ocp-release:4.18.28-multi</code></pre><p dir="ltr"><span>Wait for 10 to 15 minutes to have the hosted cluster created. While waiting, you can check the status on the GUI of the multicluster engine for Kubernetes (aka hosting) cluster (Cluster B).</span></p><p dir="ltr"><span>We can check the status of the hosted cluster from Red Hat Advanced Cluster Management (Cluster A) and Nodepools status from the multicluster engine for Kubernetes cluster (Cluster B) as in the following.</span></p><pre><code class="language-plaintext">$ oc --kubeconfig=kubeconfig-acm get managedcluster&nbsp;
NAME&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; HUB ACCEPTED &nbsp; MANAGED CLUSTER URLS &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; JOINED &nbsp; AVAILABLE &nbsp; AGE
local-cluster &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; true &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; https://api.acm.example.com:6443 &nbsp; True &nbsp; &nbsp; True&nbsp; &nbsp; &nbsp; &nbsp; 46h
mce-with-ocpv &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; true &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; https://api.mce.example.com:6443 &nbsp; True &nbsp; &nbsp; True&nbsp; &nbsp; &nbsp; &nbsp; 4h10m
mce-with-ocpv-mce-hc1 &nbsp; true &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; https://192.168.34.205:6443&nbsp; &nbsp; &nbsp; &nbsp; True &nbsp; &nbsp; True&nbsp; &nbsp; &nbsp; &nbsp; 4h2m</code></pre><p dir="ltr"><span>Notice that the hosted cluster </span><code><span>mce-hc1</span></code><span> is prefixed with </span><code><span>mce-with-ocpv</span></code><span> as it was configured to import automatically with the name of the managed cluster.</span></p><pre><code class="language-plaintext">$ oc --kubeconfig=kubeconfig-mce get managedcluster&nbsp;
NAME&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; HUB ACCEPTED &nbsp; MANAGED CLUSTER URLS &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; JOINED &nbsp; AVAILABLE &nbsp; AGE
local-cluster &nbsp; true &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; https://api.mce.example.com:6443 &nbsp; True &nbsp; &nbsp; True&nbsp; &nbsp; &nbsp; &nbsp; 4h26m
mce-hc1 &nbsp; &nbsp; &nbsp; &nbsp; true &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; https://192.168.34.205:6443&nbsp; &nbsp; &nbsp; &nbsp; True &nbsp; &nbsp; True&nbsp; &nbsp; &nbsp; &nbsp; 4h10m

$ oc --kubeconfig=kubeconfig-mce get hostedcluster -n clusters
NAME &nbsp; &nbsp; &nbsp; VERSION &nbsp; KUBECONFIG &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; PROGRESS&nbsp; &nbsp; AVAILABLE &nbsp; PROGRESSING &nbsp; MESSAGE
mce-hc1 &nbsp; 4.18.28 &nbsp; mce-hc1-admin-kubeconfig &nbsp; Completed &nbsp; True&nbsp; &nbsp; &nbsp; &nbsp; False &nbsp; &nbsp; &nbsp; &nbsp; The hosted control plane is available

$ oc --kubeconfig=kubeconfig-mce get vmi -n clusters-mce-hc1
NAME&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; AGE&nbsp; &nbsp; PHASE &nbsp; &nbsp; IP &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; NODENAME&nbsp; &nbsp; &nbsp; READY
mce-hc1-j6vtm-8c5kd &nbsp; 4h4m &nbsp; Running &nbsp; 10.130.0.158 &nbsp; mce-worker1 &nbsp; True
mce-hc1-j6vtm-gnpc9 &nbsp; 4h5m &nbsp; Running &nbsp; 10.130.0.157 &nbsp; mce-worker1 &nbsp; True</code></pre><p dir="ltr"><span>We can see the Red Hat Advanced Cluster Management Cluster A in Figure 6.</span></p><figure class="rhd-u-has-filter-caption">

<figure class="media media--type-image media--view-mode-article-content rhd-c-figure">
  
        <a href="https://developers.redhat.com/sites/default/files/image4_105.png"><img alt="Cluster view of Cluster A." height="199" src="https://developers.redhat.com/sites/default/files/styles/article_floated/public/image4_105.png?itok=Yw8jP-fY" width="600" />

</a>

  </figure>

<figcaption class="rhd-c-caption">Figure 6: Cluster view of Cluster A.</figcaption>
</figure>
<p dir="ltr"><span>Finally, we can check the hosted cluster and run commands.</span></p><pre><code class="language-plaintext">$ export KUBECONFIG=kubeconfig-mce ; hcp create kubeconfig --name mce-hc1 --namespace clusters &gt; kubeconfig-mce-hc1

$ oc --kubeconfig=kubeconfig-mce-hc1&nbsp;get nodes
NAME&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; STATUS &nbsp; ROLES&nbsp; &nbsp; AGE&nbsp; &nbsp; VERSION
mce-hc1-j6vtm-8c5kd &nbsp; Ready&nbsp; &nbsp; worker &nbsp; 4h7m &nbsp; v1.31.13
mce-hc1-j6vtm-gnpc9 &nbsp; Ready&nbsp; &nbsp; worker &nbsp; 4h8m &nbsp; v1.31.13

$ oc --kubeconfig=kubeconfig-mce-hc1 whoami --show-console
https://console-openshift-console.apps.mce-hc1.apps.mce.example.com</code></pre><p dir="ltr"><span>Notice that the console is pointing to a wildcard DNS of the Cluster B.</span></p><pre><code class="language-plaintext">$ oc --kubeconfig=kubeconfig-mce-hc1 whoami --show-server
https://192.168.34.205:6443</code></pre><p dir="ltr"><span>Notice that the API address is pointing to the IP address provided by the MetalLB range we previously configured.</span></p><p dir="ltr"><strong>The following lists pros &amp; cons of this topology:</strong></p><p dir="ltr"><span>Pros:</span></p><ul><li dir="ltr"><span>The most stable and scalable production design</span></li><li dir="ltr"><span>Hub is lightweight and secure</span></li><li dir="ltr"><span>Hosting resources scale independently</span></li><li dir="ltr"><span>Enterprise teams can align with operational boundaries</span></li></ul><p dir="ltr"><span>Cons:</span></p><ul><li dir="ltr"><span>Slightly more operational complexity</span></li><li dir="ltr"><span>HostedClusters must be imported manually (unless automated)</span></li><li dir="ltr"><span>Networking dependencies between clusters</span></li></ul><h2 dir="ltr"><span>Wrap up</span></h2><p dir="ltr"><span>In this installment, we separated Red Hat Advanced Cluster Management (hub) and the multicluster engine for Kubernetes/hosted control plane/OpenShift Virtualization (management and hosting) across two clusters. This pattern ensures clean isolation, predictable scaling, and clear team responsibilities. But some customers operate on yet another level of scale and flexibility: they split control-plane hosting from worker-node hosting.</span></p><p dir="ltr"><span>In the next installment, we'll discuss where an advanced architecture runs:</span></p><ul><li dir="ltr"><span>Red Hat Advanced Cluster Management Hub on Cluster A</span></li><li dir="ltr"><span>Multicluster engine for Kubernetes + hosted control plane on Cluster B</span></li><li dir="ltr"><span>OpenShift Virtualization (NodePool VMs only) on Cluster C</span></li><li dir="ltr"><span>NodePools live on Cluster C even though control planes run on Cluster B</span></li></ul><p dir="ltr"><span>This allows incredible flexibility in worker VM placement and multi-zone hosting strategies. Stay tuned for Part 3.</span></p>

The post <a href="https://developers.redhat.com/articles/2026/04/27/deploy-hosted-control-planes-openshift-virtualization-split-hub" title="Deploy hosted control planes with OpenShift Virtualization: Split hub">Deploy hosted control planes with OpenShift Virtualization: Split hub</a> appeared first on <a href="https://developers.redhat.com/blog" title="Red Hat Developer">Red Hat Developer</a>.
<br /><br />

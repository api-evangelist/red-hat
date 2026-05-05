---
title: "Build .NET container images with Tekton"
url: "https://developers.redhat.com/articles/2026/04/28/build-net-container-images-tekton"
date: "Tue, 28 Apr 2026 07:16:21 +0000"
author: "Tom Deseyn"
feed_url: "https://developers.redhat.com/blog/feed"
---
<p><a href="https://tekton.dev/">Tekton</a> is a Kubernetes-native CI/CD framework that lets you define pipelines as Kubernetes resources. If you're using Tekton to build and deploy .NET applications, the <a href="https://artifacthub.io/packages/tekton-task/redhat-tekton-tasks/dotnet-publish-image">dotnet-publish-image</a> task lets you build and push container images directly using the .NET SDK without writing a Dockerfile.</p><p>In this post, we'll walk through setting up a minimal Tekton pipeline that clones a .NET application from Git, builds a container image, and pushes it to <a href="https://quay.io">quay.io</a>.</p><h2>How the task works</h2><p>The <code>dotnet-publish-image</code> Tekton task uses .NET's <a href="https://learn.microsoft.com/en-us/dotnet/core/containers/publish-as-container">built-in container publishing</a> support to build and push container images. This means no Dockerfile or separate build tool is required.</p><p>The task takes a few key parameters:</p><ul><li><code>SDK_IMAGE</code>: The .NET SDK image to use for building (for example, <code>registry.access.redhat.com/dotnet/sdk:10.0</code>).</li><li><code>PROJECT</code>: Path to the <code>.csproj</code> file in the source workspace.</li><li><code>IMAGE_NAME</code>: The target image repository.</li><li><code>BASE_IMAGE</code> (optional): The base image for the application container. When the tag is omitted, it is inferred from the project's target framework.</li></ul><p>By default, the .NET SDK uses Microsoft's base images. To use <a href="https://developers.redhat.com/articles/2025/12/01/what-you-need-know-about-red-hats-net-container-images">Red Hat's .NET container images</a> instead, set <code>BASE_IMAGE</code> to the appropriate image for your application type. For example, for ASP.NET Core applications use <code>registry.access.redhat.com/dotnet/aspnet</code>. For non-web applications use <code>registry.access.redhat.com/dotnet/runtime</code>.</p><h2>Install the tasks</h2><p>Our pipeline uses two tasks: <a href="https://artifacthub.io/packages/tekton-task/tekton-tasks/git-clone">git-clone</a> to fetch the source code and <a href="https://artifacthub.io/packages/tekton-task/redhat-tekton-tasks/dotnet-publish-image">dotnet-publish-image</a> to build and push the container image. Install them using the Tekton CLI:</p><pre><code class="language-plaintext">tkn hub install task git-clone
tkn hub install task dotnet-publish-image --type artifact --from redhat-tekton-tasks</code></pre><h2>Set up the image repository</h2><p>We will push our example application to <a href="https://quay.io">quay.io</a> using a robot account. First, create a new repository at <a href="https://quay.io/new/">quay.io</a> (for example, <code>s2i-dotnetcore-ex</code>).</p><p>Next, create a <a href="https://docs.quay.io/glossary/robot-accounts.html">robot account</a> and grant it <strong>Write</strong> permission on the repository.</p><p>Once that's in place, create a Kubernetes <code>Secret</code> with the robot account credentials. Note that the username for a robot account is your quay.io username prefixed to the robot name (e.g., <code>myuser+myrobot</code>):</p><pre><code class="language-plaintext">kubectl create secret docker-registry quay-credentials \
  --docker-server=quay.io \
  --docker-username="&lt;username&gt;+&lt;robot-name&gt;" \
  --docker-password="&lt;robot-token&gt;"</code></pre><p>Finally, link the secret to the <code>pipeline</code> service account so Tekton can provide it to all pipeline tasks:</p><pre><code class="language-plaintext">oc secret link pipeline quay-credentials
# or: kubectl patch serviceaccount pipeline -p '{"secrets": [{"name": "quay-credentials"}]}'</code></pre><p>Alternatively, instead of linking the secret to the <code>ServiceAccount</code>, you can pass the credentials directly to the <code>dotnet-publish-image</code> task using its optional <code>dockerconfig</code> workspace.</p><h2>The pipeline</h2><p>Here's a minimal pipeline that clones the <a href="https://github.com/redhat-developer/s2i-dotnetcore-ex">s2i-dotnetcore-ex</a> example application and builds a container image using Red Hat's .NET 10 SDK and ASP.NET Core runtime images:</p><pre><code class="language-plaintext">apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: dotnet-build-image
spec:
  params:
    - name: git-url
      type: string
      description: Git repository URL.
    - name: image-name
      type: string
      description: Target image repository.
  workspaces:
    - name: source
  tasks:
    - name: clone
      taskRef:
        name: git-clone
      params:
        - name: url
          value: $(params.git-url)
      workspaces:
        - name: output
          workspace: source
    - name: build-image
      runAfter:
        - clone
      taskRef:
        name: dotnet-publish-image
      params:
        - name: SDK_IMAGE
          value: registry.access.redhat.com/dotnet/sdk:10.0
        - name: PROJECT
          value: app/app.csproj
        - name: IMAGE_NAME
          value: $(params.image-name)
        - name: BASE_IMAGE
          value: registry.access.redhat.com/dotnet/aspnet
      workspaces:
        - name: source
          workspace: source</code></pre><p>This pipeline has two tasks:</p><ol><li><code>clone</code> fetches the source code from Git using the <a href="https://hub.tekton.dev/tekton/task/git-clone">git-clone</a> task.</li><li><code>build-image</code> builds and pushes the container image using <code>dotnet-publish-image</code>.</li></ol><h2>Run the pipeline</h2><p>Create a <code>PipelineRun</code> to execute the pipeline:</p><pre><code class="language-plaintext">apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: dotnet-build-image-
spec:
  pipelineRef:
    name: dotnet-build-image
  params:
    - name: git-url
      value: https://github.com/redhat-developer/s2i-dotnetcore-ex
    - name: image-name
      value: quay.io/&lt;your-quay-username&gt;/s2i-dotnetcore-ex
  workspaces:
    - name: source
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi</code></pre><p>Now, start the pipeline run:</p><pre><code class="language-plaintext">kubectl create -f pipelinerun.yaml</code></pre><p>You can follow the progress with the Tekton CLI:</p><pre><code class="language-plaintext">tkn pipelinerun logs --last -f</code></pre><p>Once complete, the image will be available at <code>quay.io/&lt;your-quay-username&gt;/s2i-dotnetcore-ex</code>.</p><p>The <code>PipelineRun</code> uses a <code>volumeClaimTemplate</code> for the source workspace. Tekton automatically creates a <code>PersistentVolumeClaim</code> for each run and cleans it up when the <code>PipelineRun</code> is deleted. To control how many completed runs are retained, you can configure <a href="https://tekton.dev/docs/pruner/">Tekton Pruner</a> with history-based or time-based policies—either globally, per namespace, or scoped to specific pipelines using label selectors.</p><h2>Trigger from a Git push</h2><p>With <a href="https://tekton.dev/docs/triggers/">Tekton Triggers</a>, you can run this pipeline automatically whenever code is pushed to your repository. This involves three resource types that work together:</p><ul><li>An <code>EventListener</code> exposes an HTTP endpoint that receives webhooks. It uses a <code>TriggerBinding</code> to extract parameters and a <code>TriggerTemplate</code> to create a new <code>PipelineRun</code>.</li><li>A <code>TriggerBinding</code> extracts values from the incoming webhook payload — for example, the repository clone URL from a GitHub push event.</li><li>A <code>TriggerTemplate</code> defines a <code>PipelineRun</code> template that gets instantiated with the extracted values each time an event is received.</li></ul><p>To secure the endpoint, you can add a GitHub <a href="https://tekton.dev/docs/triggers/interceptors/">interceptor</a> to the <code>EventListener</code>. The interceptor validates the webhook signature against a shared secret, ensuring only genuine GitHub events trigger the pipeline.</p><p>Once the <code>EventListener</code> is deployed, Tekton Triggers creates a <code>Service</code> for it. You can expose it with a <code>Route</code> (on OpenShift) or an <code>Ingress</code>, and configure it as a <a href="https://docs.github.com/en/webhooks/using-webhooks/creating-webhooks">webhook in your GitHub repository settings</a> — selecting the <strong>push</strong> event. After that, every push to the repository will trigger a new pipeline run that builds and pushes an updated container image.</p><h2>Conclusion</h2><p>The <a href="https://artifacthub.io/packages/tekton-task/redhat-tekton-tasks/dotnet-publish-image"><code>dotnet-publish-image</code></a> task provides a straightforward way to build .NET container images in Tekton pipelines. It uses the .NET SDK's built-in container support, so there's no need to write or maintain Dockerfiles. Combined with <a href="https://developers.redhat.com/articles/2025/12/01/what-you-need-know-about-red-hats-net-container-images">Red Hat's .NET container images</a>, this task gives you a minimal, production-ready pipeline for building and publishing .NET application images.</p><p>The task supports additional features beyond what we covered here, such as passing MSBuild properties and environment variables. For the full list of parameters and usage details, see the <a href="https://github.com/redhat-developer/dotnet-tekton-tasks/blob/main/docs/task-dotnet-publish-image.md">task documentation</a>.</p>
The post <a href="https://developers.redhat.com/articles/2026/04/28/build-net-container-images-tekton" title="Build .NET container images with Tekton">Build .NET container images with Tekton</a> appeared first on <a href="https://developers.redhat.com/blog" title="Red Hat Developer">Red Hat Developer</a>.
<br /><br />

# ArgoPacks
Argo based Serverless like CI system for Kubernetes

# ArgoPacks

Areas: Kubernetes, Software Delivery
Creator: Marcin Karkocha
Tags: Argo, CI, CRD, K8s

# Description

Idea of creating Argo Workflow and Argo Events based CI framework for Kubernetes with global reusable objects for most parts of process.

## Idea

ArgoPacks will be easy to use definitions that take care of building our applications into container registry.

```yaml
apiVersion: packs.argoproj.io/v1beta1
kind: Project
metadata:
  name: my-awesome-project
```

First object to remove repeatable parts of code is registry - we are configuring it in our builds, sometimes our cluster assumes more than one. Is good to define it as a object.

```yaml
apiVersion: packs.argoproj.io/v1beta1
kind: Registry
metadata: 
  name: jcr
spec:
  scpe: global
  service:
    filter: 
      name: jcr-registry
      namespace: toolbox
  credentials:
    secretFrom:
      name: jcr-creds
```

Similar way we can define repository general access. 

```yaml
apiVersion: packs.argoproj.io/v1beta1
kind: RepositoryAccount
metadata:
  name: github
  project: my-awesome-project
spec:
  scope: project
  url: github.com
  credentials:
    secretFrom:
      name: repo-creds
```

Looks now on pipeline definition: (taskrefs are argo templateRef's)

```yaml
apiVersion: packs.argoproj.io/v1beta1
kind: Builder
metadata:
  project: my-awesome-project
  name: my-go-app-builder
  buildNameTpl: "goapp-%code-revision%-%timestamp%"
spec:
  entrypoint: checkout
  steps:
    - name: checkout
      task:
        taskRef:
          name: CheckoutDef
    - name: build
      task:
        taskRef:
          name: BuildCodeDef
    - name: test
      task:
        taskRef:
          name: TestCodeDef
    - name: release-prod
      task:
        taskRef:
          name: ReleaseDef
        arguments:
          params:
            - name: form
              value: production
     - name: release-debug
       task:
         taskRef:
           name: ReleaseDef
         arguments:
           params:
             - name: form
               value: debug
```

After defining pipeline we need definition of build specification for particular builds

```yaml
apiVersion: packs.argoproj.io/v1beta1
kind: Pipeline
metadata:
  project: my-awesome-project
  name: goapp-latest
****spec:
	repository: jcr
  image: mygoapp
  tag: latest
  builder: 
		name: my-go-app-builder
		params:
			Dockerfile: golang
  code:
    git: 
      repoAccount: github
      repo: argo/argo
      revision: master
```

Now, when next build will be run we need build object created

```yaml
apiVersion: packs.argoproj.io/v1beta1
kind: Build
metadata:
  pipeline: goapp-latest
  project: my-awesome-project
```

it will automatically create params like:

- build name based on template - goapp-master-1589828589
- pipeline name
- image
    - image-url
    - image-name
    - image-tag
- creation time
- end time
- how long it takes (after pipeline is finished)
- argo workflow execution name and data from it
- notification status

For automatically schedule new builds based on some actions like code commit we need to add Webhook object.

```yaml
apiVersion: packs.argoproj.io/v1beta1
kind: Webhook
metadata:
  name: github-build
  project: my-awesome-project
spec:
  repo:
    url: github.com/argo/argo
    revision: master
  event:
    pipeline:
      name: goapp-latest
    task:
       taskRef: slack-message
```

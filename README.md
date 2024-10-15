# BAREMETAL K8S OAKDEW

### Run in local

- Actualiza el archivo /etc/hosts
  Con este enfoque, solo necesitas una línea en tu archivo /etc/hosts:

`192.168.1.230 oakdew.local`

### Install MetalLb

Configure `kube-proxy`:

```
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

### Install NGINX Ingress Controller

```
helm upgrade --install ingress-nginx ingress-nginx \
 --repo https://kubernetes.github.io/ingress-nginx \
 --namespace ingress-nginx --create-namespace
```

You can watch the status by running 'kubectl get service --namespace ingress-nginx ingress-nginx-controller --output wide --watch'

An example Ingress that makes use of the controller:

```
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls
```

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

```
  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

## Gitlab

Update file `gitlab.rb` to configure Redis:

```
gitlab_rails['redis_host'] = 'redis'
gitlab_rails['redis_port'] = 6379
gitlab_rails['redis_ssl'] = false
gitlab_rails['redis_password'] = nil
gitlab_rails['redis_database'] = 0
gitlab_rails['redis_enable_client'] = true
```

Then restart the Gitlab:

```
gitlab-ctl reconfigure
gitlab-ctl restart
```

### 500 error when accessing CI/CD in a project

In Gitlab Pod Terminal run in `gitlab-rails console`:

```
projects = Project.all

project = Project.find(<PROJECT_ID>)

if project
  project.update(runners_token_encrypted: nil)
else
  puts "Project not found."
end
```

Same erro for CI/CD in admin:

```
gitlab-rails dbconsole

-- Clear project tokens
UPDATE projects SET runners_token = null, runners_token_encrypted = null;

-- Clear group tokens
UPDATE namespaces SET runners_token = null, runners_token_encrypted = null;

-- Clear instance tokens
UPDATE application_settings SET runners_registration_token_encrypted = null;

-- Clear runner tokens
UPDATE ci_runners SET token = null, token_encrypted = null;
https://docs.gitlab.com/ee/raketasks/backup_restore.html#reset-runner-registration-tokens
```

Edit:

After clearing existing pipeline jobs (see above), I could still not open the ci settings page for some migrated projects where I had set environment variables. In this case try to remove them:

```
gitlab-rails dbconsole
SELECT * FROM ci_variables;
DELETE FROM ci_variables WHERE project_id='XX';
```

Gitlab runner have to be installed in the project through manifest file and the runner has to be registered in the gitlab project by opening terminal and running the command. CONFIG_VALUES_FILE is in gitlab folder:

```
helm repo update gitlab

helm install --namespace <NAMESPACE> --name gitlab-runner -f <CONFIG_VALUES_FILE> gitlab/gitlab-runner
```

Add to the runner configmap:

```

[[runners]]
  name = "docker-runner"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.docker]
    tls_verify = false
    image = "docker:20.10.7"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
```

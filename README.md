# FogProtect Usecase
This is an application that demonstrates the use of [fybrik](https://github.com/fybrik/fybrik) and 
leverages it to control the data flow between a webserver and a data server, where the webserver 
sends HTTP requests to data server and the data server responds accordingly.
## Environment Build
1) Follow the steps of the QuickStart Guide in: https://fybrik.io/v0.4/get-started/quickstart/.  
Displayed here for convenience:  
   1) ```
      kind create cluster --name kind-cluster
   2) ```
      helm repo add jetstack https://charts.jetstack.io 
      helm repo add hashicorp https://helm.releases.hashicorp.com 
      helm repo add fybrik-charts https://fybrik.github.io/charts 
      helm repo update
   3) ```
      helm install cert-manager jetstack/cert-manager \
      --namespace cert-manager \
      --version v1.2.0 \
      --create-namespace \
      --set installCRDs=true \
      --wait --timeout 120s
   4) ```
      git clone https://github.com/fybrik/fybrik.git
      cd fybrik
      helm dependency update charts/vault
      helm install vault charts/vault --create-namespace -n fybrik-system \
      --set "vault.injector.enabled=false" \
      --set "vault.server.dev.enabled=true" \
      --values charts/vault/env/dev/vault-single-cluster-values.yaml
      kubectl wait --for=condition=ready --all pod -n fybrik-system --timeout=120s
   5) ```
      helm install fybrik-crd charts/fybrik-crd -n fybrik-system --wait
      helm install fybrik charts/fybrik --set global.tag=latest -n fybrik-system --wait
   6) Change the current directory to the previous directory:
   ```shell
   cd ..
   ```
2) Create a namespace that will contain the application and the backend data server:  
    ```shell
    kubectl create namespace fogprotect
    kubectl config set-context --current --namespace=fogprotect
    ```

## Deployment
1) Deploy the backend data server:
    ```shell
    helm chart pull ghcr.io/robshahla/backend-server-chart:v0.0.1
    helm chart export --destination=./tmp ghcr.io/robshahla/backend-server-chart:v0.0.1
    helm install rel1-backend-server ./tmp/backend_server
    ```

2) Create the assets:
    ```shell
    wget assets...
    kubectl apply -f assets/
    ```  
3) Create all of the secrets under the directory `fog-protect/secrets` in the 
`forprotect` namespace (the current context of `kubectl`), and afterwards create the `jwt_key_secret.yaml` in 
the `fybrik-blueprints` namespace:  
    ```shell
    kubectl apply -f secrets/
    kubectl apply -n fybrik-blueprints -f secrets/jwt_key_secret.yaml
    ```

4) Create the RBAC for the fybrik manager so that the manager can list the assets and other resources:  
    ```shell
    wget rbac...
    kubectl apply -f fybrik-system-manager-rbac.yaml -n fybrik-system
    ```

5) Deploy the filter pod:
    ```shell
    kubectl apply -f rest-read-module.yaml -n fybrik-system
    kubectl apply -f rest-read-application.yaml
    ```

6) Wait a couple of seconds after the last step, and then create a port-forwarding to the filter service:  
    ```shell
    kubectl -n fybrik-blueprints port-forward svc/rest-read 5559:5559
    ```

7) Deploy the pod of the GUI:
    ```shell
    helm chart pull ghcr.io/robshahla/factory-gui-chart:v0.0.1
    helm chart export --destination=./tmp ghcr.io/robshahla/factory-gui-chart:v0.0.1
    helm install rel1-factory-gui ./tmp/factory_gui
    kubectl wait --for=condition=ready --all pod --timeout=120s
    ```

8) Create a port-forwarding to the GUI service:
    ```shell
    kubectl port-forward svc/factory-gui 3001:3000
    ```  

9) Open a browser and go to: `http://127.0.0.1:3001` to use the GUI.  

## Development
1) Clone this repository:  
    ```shell
    git clone git@github.com:fybrik/fog-protect.git
    ```

2) Follow the instructions in https://github.com/fybrik/hello-world-read-module#installation 
under the `Installation` section, brought here for convenience:  
   -  After modifying the values in the Makefile as required in the link above, 
      **make sure that the current directory is `fog-protect/`**. Invoke:  
      ```shell
      make docker-build
      make docker-push
      make helm-login
      make helm-verify
      make helm-chart-push
      ```
    **Improtant: make sure that the repositories where the docker image and the helm chart were pushed 
       are public**  
3) Apply the RBAC for the fybrik manager:  
    ```shell
    kubectl apply -f fybrik-system-manager-rbac.yaml -n fybrik-system
    ```
4) Apply all of the assets, along with the `secret` under the directory `fog-protect/assets` in the 
`forprotect` namespace (the current context of `kubectl`):  
    ```shell
    kubectl apply -f assets/
    ```

5) Create all of the secrets under the directory `fog-protect/secrets` in the 
`forprotect` namespace (the current context of `kubectl`), and afterwards create the `jwt_key_secret.yaml` in 
the `fybrik-blueprints` namespace:  
    ```shell
    kubectl apply -f secrets/
    kubectl apply -n fybrik-blueprints -f secrets/jwt_key_secret.yaml
    ```
    **Note:** the secret `secrets/jwt_key_secret.yaml` contains the secret key used as the authentication key for the 
    JWT used between the rest filter and the frontend GUI, in order to change the key, you can invoke:  
   - ```shell
     echo -n '<your_key>' | base64
     ```
     Once you get the base64 encoding of your key, modify the value of `data.jwt_key` in `jwt_key_secret.yaml`. 
     In order for the change to take effect, the GUI and the rest-read pods need to be restarted.  

6) Apply the module in the `fybrik-system` namespace:  
    ```shell 
    kubectl apply -f rest-read-module.yaml -n fybrik-system
    ```
7) Apply the application in the `fogprotect` namespace (current `kubectl` context):
    ```shell
    kubectl apply -f rest-read-application.yaml
    ```
8) Wait a couple of seconds after the last step, and then create the port-forwarding:  
    ```shell
    kubectl -n fybrik-blueprints port-forward svc/rest-read 5559:5559
    ```
9) To build the image for the GUI and push it to the public images registry specified in the makefile, invoke the 
following:
    ```shell
    make DOCKER_IMG_NAME=factory-gui DOCKER_FILE=./gui/Dockerfile DOCKER_IMG_CONTEXT=./gui docker-all
    make DOCKER_CHART_IMG_NAME=factory-gui-chart DOCKER_IMG_NAME=factory-gui DOCKER_FILE=./gui/Dockerfile CHART_PATH=./gui/helm helm-verify
    make DOCKER_CHART_IMG_NAME=factory-gui-chart DOCKER_IMG_NAME=factory-gui DOCKER_FILE=./gui/Dockerfile CHART_PATH=./gui/helm helm-chart-push
    ```
   - To deploy the GUI pod invoke:
       ```shell
       make DOCKER_CHART_IMG_NAME=factory-gui-chart DOCKER_IMG_NAME=factory-gui DOCKER_FILE=./gui/Dockerfile CHART_PATH=./gui/helm helm-install
       kubectl wait --for=condition=ready --all pod --timeout=120s
       ```
     Create a port-forwarding to the GUI service:
       ```shell
       kubectl port-forward svc/factory-gui 3001:3000
       ```
10) To build the image for the backend server and push it to the public image registry specified in the makefile, invoke the 
following:
    ```shell
    make DOCKER_IMG_NAME=backend-server DOCKER_FILE=./python/backend/Dockerfile DOCKER_IMG_CONTEXT=./python/backend docker-all
    make DOCKER_CHART_IMG_NAME=backend-server-chart DOCKER_IMG_NAME=backend-server DOCKER_FILE=./python/backend/Dockerfile CHART_PATH=./python/backend/helm helm-verify
    make DOCKER_CHART_IMG_NAME=backend-server-chart DOCKER_IMG_NAME=backend-server DOCKER_FILE=./python/backend/Dockerfile CHART_PATH=./python/backend/helm helm-chart-push
    ```
    - To deploy the backend server pod invoke:
      ```shell
      make DOCKER_CHART_IMG_NAME=backend-server-chart DOCKER_IMG_NAME=backend-server DOCKER_FILE=./python/backend/Dockerfile CHART_PATH=./python/backend/helm helm-install
      ```  
      
    Another option instead of pushing the image to a public image registry is to build it locally and load it into the local
    cluster, to do that invoke the following:
     ```shell
     docker build -t backend_server:v1 .
     kind load docker-image backend_server:v1 --name kind-cluster
     helm install backend-service backend_server-0.1.0.tgz
     ```

11) Open a browser and go to: `http://127.0.0.1:3001` to use the GUI.  
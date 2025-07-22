# Manage your Kubernetes Environment

## Setup local access to a remote kubernetes cluster

#### **Prerequisites**

Before embarking on this journey, ensure you have the following prerequisites in place:

1. **A running Kubernetes cluster:** Confirm that your remote [Kubernetes cluster](https://www.notion.so/How-to-Build-a-Kubernetes-3-Node-Cluster-cc8f6049b6544509b68fb44b90986a78?pvs=21) is operational, deployed using kubeadm.
2. **kubectl installed locally:** Install the Kubernetes command-line tool, [kubectl](https://kubernetes.io/docs/tasks/tools/), on your local machine.
3. SSH Access from the local machine to the kube-master.local server

### Setup Local Server

**On local machine**

Create a folder called .kube under your local home folder

```jsx
cd
mkdir .kube 
```

### Obtain Cluster Config

**on local machine**

The file admin.conf is needed from /etc/kubernetes/ o the kube-master.local server

Assuming the remote user on the kube-master.local server is user1 and your local user can ssh to kube-master.local as that user then the following commands should be used

```jsx
scp user1@kube-master.local:/etc/kubernetes/admin.conf .kube/config
```

**Note:**

**Using this method, any user using the local machine will need to pull down the config file**

### Check .kube/config

**on local machine**

check the config file looks like is should

```jsx
apiVersion: v1
clusters:
- cluster:sgsOQVFFTEJRQXdGVEVUTUJFR0EffhssdfghfhsdfgsgGN6QWVGdzB5TkRBMk1UZ3hNREEyTkRWYUZ3MHpOREEyTVRZeE1ERXhORFZhTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUUNybmJsem12dnV6UmVDYnUrSThHQ2VleWk3Tkc0eGFlQnNrcTlzUDBYM3ptUW1WZDlsQkdoNzRmeEEKSkRJOGJJSFArK2I2RTJJSW9OMXo4ODFqWXNSNkVpc0UzMkFpUmVyM2FHbEVtM2Z5N29VY1VGcFVxUG9ZYldzQQpUZlhWTlRHWVJBZmRhL25JZU4vbVZQZmNYU215WWoreEpKSnJwQzlaZDk3UzY4NzF3KzFhRTBmNnhUN1YrU2EwCjhGWTZpem5udWowVU5sQWpRcG9HL2xWdTZaYUlvR1Npdm1rT0NWLzNtaFpKZ1FEVFJpb2dsUG9rTGNkeFNHcm8KR0dHL3dJMXB0b2JJMVA1cXUyVmQrY3pIQmtMOXhodm1jMHVoa2dvTmhIMVowa2YxbERBSlk5clRtT0ZBMVNuMQphL00yWTNFS1BJNFgwdjF2UFEwSzNoeWVhbDV4QWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJUVHY2SWxTQU5CZVBuU3llM0hTZVRvRVdDTDZqQVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQkx1MW5nVDd6dApwZWZMMDVSRERQbEljWWd6NWVjcU9hb2FzMkd5SEtkZHhXK1IwdDRXanZIUVFoS3pzOE5JVW1GcTlndm80dUxECkx6dDRGTWFOL3RhUEsyM0pPVUp1RDhjNE1TdmpZenZCK2NOb2FIQThjWmRodXBIYy9ydzFJQUhaSWxaZ3M1NjEKK0VVUzlwNDd6cU1BbHQ1QmxBREwreGxLUExuZEdzSzhBTVJRMVAzVkNxS1QyN2dieEZnRFYwU3VWRDRoUEF4UApZaVRKWkhTaTVpbTg1RFFsSTdiQ0hseTIyblg3UXZuRjNCNngvWjl5YjdjSUJYZXZpcEhTaHFGVUlrOXFCeFE2CjE0dnFGVHZFYVZRU1FxcEgyZUh1R21WbXJYekpjMytUWENzQ1BENjd0aGUxZGdMblBEaGRTWDcvY1JBS0NWa0gKaVMwMHBBUUducWM2Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: <https://k8s-master:6443>
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDfhjldfhjdfghjkl;dghjkl;szdg;hjzg;hjzdg;hjklzgZ0F3SUJBZ0lJSlFqYzlrb2hqZHN3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TkRBMk1UZ3hNREEyTkRWYUZ3MHlOVEEyTVRneE1ERXhORGRhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXQ5c2dNeGt3Z0h1UTYrcU0KTTZ2VmlBY2Vmd1BCK0NGZzRkeVltelYxcEsrOEdBRWZ2Qjg1bmNhT1lVZzNPZHVPZXQ4V2JBcnBXM3JqZXVIRgpXQzlvd2hBRTVEeGNEK1h0TlZVaVZpelB1VkxMOEZ0REtCb1hwV3h5SFdIK2lNNkZFRmVrbVdiVDFlY3NJaXVMCkFlNWN3QkUrNitFa2VxUUtpWC9VdE9mNDlGaVJtdmhaS1BCYVlsZ1pjREdFYTVoeDNRM3JxYjcxYVB4Z0w3YUQKdURGNnpSRE5NUkt4VVZ1TjFROFJIei9Ia1FvNVNaUE1Lc1JtYzJ6MHpHN3gwWEVVM0s2cXA4UjJmMEFaT25LegpjdWFITkp3MFV2T25nYlpESUxQYTlXQ3dFclJKb2xxa0E2Zk1tRFRWY2dtYllYN0ZIWFMrZkg1YXB4engvK0Y4CkhDazlBUUlEQVFBQm8xWXdWREFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RBWURWUjBUQVFIL0JBSXdBREFmQmdOVkhTTUVHREFXZ0JUVHY2SWxTQU5CZVBuU3llM0hTZVRvRVdDTAo2akFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBQ0g3dUU2ZW1lS2hNWmI3TENuOFhuVVdreVJ4SFVvRkZEQ2FoClluZzJ6WmRxM1BUaEpCR1ZyQTZhQ0ZqMEw1SDRyMFc0Z3FzZTRxOVhJMjQzaFJBaVpYRlh0enZZeWxYaVROYWsKcHhiMEk1YVc0L21sL3I2Qy9qaUc5MUhrV0pKQ0EwWGtseEdtb0duSzdIaHBacHB6WE1QdEV5azBXTVJmbUtPdgpudDBqNytVaUlXQUcyMUVhL210bE5Bc2k4UlZVM2lXaTlWZ0J4TlVacG9UOStlNldkdmFzTGJwZmE0Z3V0Q3BFCnA0dHFVVDVJRmFIa3NrbXZVUkwweUhlUW9yRS9GZ05XcHBUcmM2MmFhMlFwaGhXbGl5alRuaVYvWVZ6TGViK3UKdHY5bzF4d0ZwV2RkTW93dmdjU2ZnQWkxVUdidjlHdnFGQTZWa3oxOUErUHEyNmVVVnc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFgshjl;gsjkl;sdghjkl;sdgkhjlsdfghjkl;sdfghjkl;asfgVWlWaXpQdVfghf
```

### Update kubectl config

**on local machine**

Setup an export

```jsx
export KUBECONFIG=~/.kube/config
```

### Test kubectl command

**on local machine**

run the following command as local user (user1)

```jsx
kubectl config view
```

You should see something similar

```jsx
apiVersion: v1
clusters:
- cluster:
certificate-authority-data: DATA+OMITTED
server: <https://k8s-master:6443>
name: kubernetes
contexts:
- context:
cluster: kubernetes
user: kubernetes-admin
name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
user:
client-certificate-data: DATA+OMITTED
client-key-data: DATA+OMITTED
```

### Done

## Setup Lens

### What is lens?

Lens is a cross platform GUI from Mirantis which is used to view and interact with Kubernetes clusters. While learning kubernetes it is highly useful for seeing what is going on within your (test) cluster.

[https://k8slens.dev/](https://k8slens.dev/)

### Launch Lens

**on local machine**

Install Lens on your preferred platform

Run through the Lens setup

Setup a Lens Cloud account

### Add your Kubernetes cluster to Lens

**on local machine**

Once Lense is open, click on **File → Add Cluster**

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/9367f979-6ae8-4be5-b285-d0a67abd79b1/Untitled.png)

Paste the contents of you .kube/config file

```jsx
apiVersion: v1
clusters:
- cluster:sgsOQVFFTEJRQXdGVEVUTUJFR0EffhssdfghfhsdfgsgGN6QWVGdzB5TkRBMk1UZ3hNREEyTkRWYUZ3MHpOREEyTVRZeE1ERXhORFZhTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUUNybmJsem12dnV6UmVDYnUrSThHQ2VleWk3Tkc0eGFlQnNrcTlzUDBYM3ptUW1WZDlsQkdoNzRmeEEKSkRJOGJJSFArK2I2RTJJSW9OMXo4ODFqWXNSNkVpc0UzMkFpUmVyM2FHbEVtM2Z5N29VY1VGcFVxUG9ZYldzQQpUZlhWTlRHWVJBZmRhL25JZU4vbVZQZmNYU215WWoreEpKSnJwQzlaZDk3UzY4NzF3KzFhRTBmNnhUN1YrU2EwCjhGWTZpem5udWowVU5sQWpRcG9HL2xWdTZaYUlvR1Npdm1rT0NWLzNtaFpKZ1FEVFJpb2dsUG9rTGNkeFNHcm8KR0dHL3dJMXB0b2JJMVA1cXUyVmQrY3pIQmtMOXhodm1jMHVoa2dvTmhIMVowa2YxbERBSlk5clRtT0ZBMVNuMQphL00yWTNFS1BJNFgwdjF2UFEwSzNoeWVhbDV4QWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJUVHY2SWxTQU5CZVBuU3llM0hTZVRvRVdDTDZqQVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQkx1MW5nVDd6dApwZWZMMDVSRERQbEljWWd6NWVjcU9hb2FzMkd5SEtkZHhXK1IwdDRXanZIUVFoS3pzOE5JVW1GcTlndm80dUxECkx6dDRGTWFOL3RhUEsyM0pPVUp1RDhjNE1TdmpZenZCK2NOb2FIQThjWmRodXBIYy9ydzFJQUhaSWxaZ3M1NjEKK0VVUzlwNDd6cU1BbHQ1QmxBREwreGxLUExuZEdzSzhBTVJRMVAzVkNxS1QyN2dieEZnRFYwU3VWRDRoUEF4UApZaVRKWkhTaTVpbTg1RFFsSTdiQ0hseTIyblg3UXZuRjNCNngvWjl5YjdjSUJYZXZpcEhTaHFGVUlrOXFCeFE2CjE0dnFGVHZFYVZRU1FxcEgyZUh1R21WbXJYekpjMytUWENzQ1BENjd0aGUxZGdMblBEaGRTWDcvY1JBS0NWa0gKaVMwMHBBUUducWM2Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: <https://k8s-master:6443>
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDfhjldfhjdfghjkl;dghjkl;szdg;hjzg;hjzdg;hjklzgZ0F3SUJBZ0lJSlFqYzlrb2hqZHN3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TkRBMk1UZ3hNREEyTkRWYUZ3MHlOVEEyTVRneE1ERXhORGRhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXQ5c2dNeGt3Z0h1UTYrcU0KTTZ2VmlBY2Vmd1BCK0NGZzRkeVltelYxcEsrOEdBRWZ2Qjg1bmNhT1lVZzNPZHVPZXQ4V2JBcnBXM3JqZXVIRgpXQzlvd2hBRTVEeGNEK1h0TlZVaVZpelB1VkxMOEZ0REtCb1hwV3h5SFdIK2lNNkZFRmVrbVdiVDFlY3NJaXVMCkFlNWN3QkUrNitFa2VxUUtpWC9VdE9mNDlGaVJtdmhaS1BCYVlsZ1pjREdFYTVoeDNRM3JxYjcxYVB4Z0w3YUQKdURGNnpSRE5NUkt4VVZ1TjFROFJIei9Ia1FvNVNaUE1Lc1JtYzJ6MHpHN3gwWEVVM0s2cXA4UjJmMEFaT25LegpjdWFITkp3MFV2T25nYlpESUxQYTlXQ3dFclJKb2xxa0E2Zk1tRFRWY2dtYllYN0ZIWFMrZkg1YXB4engvK0Y4CkhDazlBUUlEQVFBQm8xWXdWREFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RBWURWUjBUQVFIL0JBSXdBREFmQmdOVkhTTUVHREFXZ0JUVHY2SWxTQU5CZVBuU3llM0hTZVRvRVdDTAo2akFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBQ0g3dUU2ZW1lS2hNWmI3TENuOFhuVVdreVJ4SFVvRkZEQ2FoClluZzJ6WmRxM1BUaEpCR1ZyQTZhQ0ZqMEw1SDRyMFc0Z3FzZTRxOVhJMjQzaFJBaVpYRlh0enZZeWxYaVROYWsKcHhiMEk1YVc0L21sL3I2Qy9qaUc5MUhrV0pKQ0EwWGtseEdtb0duSzdIaHBacHB6WE1QdEV5azBXTVJmbUtPdgpudDBqNytVaUlXQUcyMUVhL210bE5Bc2k4UlZVM2lXaTlWZ0J4TlVacG9UOStlNldkdmFzTGJwZmE0Z3V0Q3BFCnA0dHFVVDVJRmFIa3NrbXZVUkwweUhlUW9yRS9GZ05XcHBUcmM2MmFhMlFwaGhXbGl5alRuaVYvWVZ6TGViK3UKdHY5bzF4d0ZwV2RkTW93dmdjU2ZnQWkxVUdidjlHdnFGQTZWa3oxOUErUHEyNmVVVnc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFgshjl;gsjkl;sdghjkl;sdgkhjlsdfghjkl;sdfghjkl;asfgVWlWaXpQdVfghf
```

like this

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/7840b624-9dac-4e12-bc0c-3bb3fbf395b3/Untitled.png)

Click on Add Cluster

After a few seconds your cluster should be visible

### View

Your initial view will look as follows

Note:

the stats will not immediately show, however once the Prometheus setup is installed, this data will be present.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/3612a1d8-7727-4f9f-b488-9bca59e2cdd7/Untitled.png)

Done

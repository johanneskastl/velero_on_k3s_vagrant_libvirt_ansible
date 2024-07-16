# velero_on_k3s_vagrant_libvirt_ansible

Vagrant-libvirt setup that creates a VM with [k3s](https://k3s.io/), the minimal
lightweight Kubernetes distribution.

On top of k3s, Ansible installs [velero](https://velero.io/).

Default OS is openSUSE Leap 15.6, but that can be changed in the Vagrantfile.
Please be aware, that this might break the Ansible provisioning.

## Vagrant

1. You need `vagrant`, obviously. And `git`. And Ansible...
1. Fetch the box, per default this is `opensuse/Leap-15.6.x86_64`, using
   `vagrant box add opensuse/Leap-15.6.x86_64`.
1. Make sure the git submodules are fully working by issuing
   `git submodule init && git submodule update`
1. Run `vagrant up`
1. Run `kubectl --kubeconfig ansible/k3s-kubeconfig get nodes` and you should
   see your server.
1. Try both NGINX ingresses by using curl. You should get a date back for both
   of them.

   ```
   curl http://nginx-pvc1.192.0.2.13.sslip.io
   curl http://nginx-pvc1.192.0.2.13.sslip.io
   ```

1. Check that the backup location is available:

   ```
   $ velero --kubeconfig ansible/k3s-kubeconfig get backup-location
   NAME    PROVIDER   BUCKET/PREFIX           PHASE       LAST VALIDATED                   ACCESS MODE   DEFAULT
   minio   aws        vagrant-libvirt-test1   Available   2024-07-16 09:41:51 +0200 CEST   ReadWrite     true
   $
   ```

1. Create a backup of the nginx-pvc1 resources and check its status:

   ```
   $ velero --kubeconfig ansible/k3s-kubeconfig backup create nginx1-backup --selector app.kubernetes.io/instance=nginx-pvc1
   [...]
   $ velero --kubeconfig ansible/k3s-kubeconfig get backup
   NAME            STATUS      ERRORS   WARNINGS   CREATED                          EXPIRES   STORAGE LOCATION   SELECTOR
   nginx1-backup   Completed   0        0          2024-07-16 09:45:58 +0200 CEST   29d       minio              app.kubernetes.io/instance=nginx-pvc1
   $ velero --kubeconfig ansible/k3s-kubeconfig describe backup nginx1-backup
   [...]
   $
   ```

1. Delete the nginx-pvc1 helm release and make sure the ingress is no longer
   responding:

   ```
   $ helm --kubeconfig ansible/k3s-kubeconfig uninstall nginx-pvc1
   [...]
   $ curl http://nginx-pvc1.192.0.2.13.sslip.io
   ```

1. Restore the nginx-pvc1 backup to another namespace:

   ```
   $ velero --kubeconfig ansible/k3s-kubeconfig restore create nginx1-restore --from-backup nginx1-backup --namespace-mappings default:nginx1-restore
   [...]
   $ velero --kubeconfig ansible/k3s-kubeconfig restore get
   $ velero --kubeconfig ansible/k3s-kubeconfig restore describe nginx1-restore
   [...]
   $
   ```

1. Try to curl the ingress again:

   ```
   $ curl http://nginx-pvc1.192.0.2.13.sslip.io
   Tue Jul 16 07:48:23 UTC 2024
   $
   ```

1. Party!

## Cleaning up

The VMs can be torn down after playing around using `vagrant destroy`. This will
also remove the kubeconfig file `ansible/k3s-kubeconfig`.

CIFS Flexvolume Plugin for Kubernetes
=====================================

Driver for [CIFS][1] (SMB, Samba, Windows Share) network filesystems as [Kubernetes volumes][2].

Background
----------

Docker containers running in Kubernetes have an ephemeral file system: Once a container is terminated, all files are gone. In order to store persistent data in Kubernetes, you need to mount a [Persistent Volume][3] into your container. Kubernetes has built-in support for network filesystems found in the most common cloud providers, like [Amazon's EBS][4], [Microsoft's Azure disk][5], etc. However, some cloud hosting services, like the [Hetzner cloud][6], provide network storage using the CIFS (SMB, Samba, Windows Share) protocol, which is not natively supported in Kubernetes.

Fortunately, Kubernetes provides [Flexvolume][7], which is a plugin mechanism enabling users to write their own drivers. There are a few flexvolume drivers for CIFS out there, but for different reasons none of them seemed to work for me. So I wrote my own, which can be found on [github.com/fstab/cifs][8].

Installing
----------

The flexvolume plugin is a single shell script named [cifs][8]. This shell scripted must be available in `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/fstab~cifs/` on the Kubernetes master and on each of the Kubernetes nodes. The directory name `fstab~cifs` will be [mapped][9] to the Flexvolume driver name `fstab/cifs`.

On the Kubernetes master and on each Kubernetes node run the following commands:

```bash
mkdir -p '/usr/libexec/kubernetes/kubelet-plugins/volume/exec/fstab~cifs'
cd '/usr/libexec/kubernetes/kubelet-plugins/volume/exec/fstab~cifs'
curl -L -O https://raw.githubusercontent.com/fstab/cifs/master/cifs
chmod 755 cifs
```

The `cifs` script requires a few executables to be available on each host system:

* `mount.cifs`, on Ubuntu this is in the [cifs-utils][10] package.
* `jq`, on Ubuntu this is in the [jq][11] package.
* `mountpoint`, on Ubuntu this is in the [util-linux][12] package.
* `base64`, on Ubuntu this is in the [coreutils][13] package.

To check if the installation was successful, run the following command:

```bash
/usr/libexec/kubernetes/kubelet-plugins/volume/exec/fstab~cifs/cifs init
```

It should output a JSON string containing `"status": "Success"`. This command is also run by Kubernetes itself when the cifs plugin is detected on the file system.

Running
-------

The plugin takes the CIFS username and password from a [Kubernetes Secret][14]. To create the secret, you first have to convert your username and password to base64 encoding:

```bash
echo -n username | base64
echo -n password | base64
```

Then, create a file `secret.yml` and use the ouput of the above commands as username and password:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cifs-secret
  namespace: default
type: fstab/cifs
data:
  username: 'ZXhhbXBsZQ=='
  password: 'bXktc2VjcmV0LXBhc3N3b3Jk'
```

Apply the secret:

```bash
kubectl apply -f secret.yml
```

You can check if the secret was installed successfully using `kubectl describe secret cifs-secret`.

Next, create a file `pod.yml` with a test pod (replace `//server/share` with the network path of your CIFS share):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: test
      mountPath: /data
  volumes:
  - name: test
    flexVolume:
      driver: "fstab/cifs"
      fsType: "cifs"
      secretRef:
        name: "cifs-secret"
      options:
        networkPath: "//server/share"
        mountOptions: "dir_mode=0755,file_mode=0644,noperm"
```

Start the pod:

```yaml
kubectl apply -f pod.yml
```

You can verify that the volume was mounted successfully using `kubectl describe pod busybox`.

Testing
-------

If everything is fine, start a shell inside the container to see if it worked:

```bash
kubectl exec -ti busybox /bin/sh
```

Inside the container, you should see the CIFS share mounted to `/data`.

[1]: https://en.wikipedia.org/wiki/Server_Message_Block
[2]: https://kubernetes.io/docs/concepts/storage/volumes/
[3]: https://kubernetes.io/docs/concepts/storage/volumes/
[4]: https://aws.amazon.com/ebs
[5]: https://azure.microsoft.com/en-us/services/storage/unmanaged-disks/
[6]: https://hetzner.cloud
[7]: https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md
[8]: https://github.com/fstab/cifs
[9]: https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md#prerequisites
[10]: https://packages.ubuntu.com/bionic/cifs-utils
[11]: https://packages.ubuntu.com/bionic/jq
[12]: https://packages.ubuntu.com/bionic/util-linux
[13]: https://packages.ubuntu.com/bionic/coreutils
[14]: https://kubernetes.io/docs/concepts/configuration/secret/

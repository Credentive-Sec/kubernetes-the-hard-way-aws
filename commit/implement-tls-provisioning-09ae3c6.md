# Implement TLS provisioning@09ae3c6

|  |  | @@ -1,5 +1,9 @@ |
| :--- | :--- | :--- |
|  |  |  variable "region" { |
|  |  |  default = "us-east-2" |
|  |  |  } |
|  |  |  |
|  |  |  provider "aws" { |
|  |  |  region = "us-east-2" |
|  |  |  region = "${var.region}" |
|  |  |  } |
|  |  |  |
|  |  |  resource "aws\_vpc" "k8s" { |
|  | @@ -196,3 +200,168 @@ resource "aws\_instance" "worker" { |  |
|  |  |  Name = "worker-${count.index}" |
|  |  |  } |
|  |  |  } |
|  |  |  |
|  |  |  locals { |
|  |  |  worker\_hostnames = "${split\(",", replace\(join\(",", aws\_instance.worker.\*.private\_dns\), ".${var.region}.compute.internal", ""\)\)}" |
|  |  |  controller\_hostnames = "${split\(",", replace\(join\(",", aws\_instance.controller.\*.private\_dns\), ".${var.region}.compute.internal", ""\)\)}" |
|  |  |  } |
|  |  |  |
|  |  |  resource "null\_resource" "tls\_ca" { |
|  |  |  provisioner "local-exec" { |
|  |  |  command = "cfssl gencert -initca tls/ca-csr.json \| cfssljson -bare tls/ca" |
|  |  |  } |
|  |  |  } |
|  |  |  |
|  |  |  resource "null\_resource" "tls\_admin" { |
|  |  |  triggers = { |
|  |  |  tls\_ca = "${null\_resource.tls\_ca.id}" |
|  |  |  } |
|  |  |  |
|  |  |  provisioner "local-exec" { |
|  |  |  command = &lt;&lt;EOF |
|  |  |  cfssl gencert \ |
|  |  |  -ca=tls/ca.pem -ca-key=tls/ca-key.pem \ |
|  |  |  -config=tls/ca-config.json \ |
|  |  |  -profile=kubernetes \ |
|  |  |  tls/admin-csr.json \ |
|  |  |  \| cfssljson -bare tls/admin |
|  |  |  EOF |
|  |  |  } |
|  |  |  } |
|  |  |  |
|  |  |  data "template\_file" "worker\_csr" { |
|  |  |  \# Terraform does not allow to use \`length\` function on computed list |
|  |  |  \# count = "${length\(aws\_instance.worker.\*.id\)}" |
|  |  |  count = "${length\(var.worker\_ips\)}" |
|  |  |  |
|  |  |  template = "${file\("tls/worker-csr.tpl.json"\)}" |
|  |  |  |
|  |  |  vars = { |
|  |  |  \# instance\_hostname = "${element\(split\(".", aws\_instance.worker.\*.private\_dns\[count.index\]\), 0\)}" |
|  |  |  instance\_hostname = "${local.worker\_hostnames\[count.index\]}" |
|  |  |  } |
|  |  |  } |
|  |  |  |
|  |  |  resource "local\_file" "worker\_csr" { |
|  |  |  \# Terraform does not allow to use \`length\` function on computed list |
|  |  |  \# count = "${length\(aws\_instance.worker.\*.id\)}" |
|  |  |  count = "${length\(var.worker\_ips\)}" |
|  |  |  |
|  |  |  content = "${data.template\_file.worker\_csr.\*.rendered\[count.index\]}" |
|  |  |  filename = "tls/worker-${count.index}-csr.json" |
|  |  |  } |
|  |  |  |
|  |  |  resource "null\_resource" "tls\_worker" { |
|  |  |  triggers = { |
|  |  |  tls\_ca = "${null\_resource.tls\_ca.id}" |
|  |  |  } |
|  |  |  |
|  |  |  \# Terraform does not allow to use \`length\` function on computed list |
|  |  |  \# count = "${length\(aws\_instance.worker.\*.id\)}" |
|  |  |  count = "${length\(var.worker\_ips\)}" |
|  |  |  |
|  |  |  provisioner "local-exec" { |
|  |  |  command = &lt;&lt;EOF |
|  |  |  cfssl gencert \ |
|  |  |  -ca=tls/ca.pem -ca-key=tls/ca-key.pem \ |
|  |  |  -config=tls/ca-config.json \ |
|  |  |  -hostname=${local.worker\_hostnames\[count.index\]},${aws\_instance.worker.\*.public\_ip\[count.index\]},${aws\_instance.worker.\*.private\_ip\[count.index\]} \ |
|  |  |  -profile=kubernetes \ |
|  |  |  ${local\_file.worker\_csr.\*.filename\[count.index\]} \ |
|  |  |  \| cfssljson -bare tls/worker-${count.index} |
|  |  |  EOF |
|  |  |  } |
|  |  |  |
|  |  |  connection { |
|  |  |  type = "ssh" |
|  |  |  user = "ubuntu" |
|  |  |  private\_key = "${tls\_private\_key.k8s.private\_key\_pem}" |
|  |  |  } |
|  |  |  |
|  |  |  provisioner "file" { |
|  |  |  source = "tls/ca.pem" |
|  |  |  destination = "/home/ubuntu/ca.pem" |
|  |  |  } |
|  |  |  |
|  |  |  provisioner "file" { |
|  |  |  source = "tls/worker-${count.index}-key.pem" |
|  |  |  destination = "worker-${count.index}-key.pem" |
|  |  |  } |
|  |  |  |
|  |  |  provisioner "file" { |
|  |  |  source = "tls/worker-${count.index}.pem" |
|  |  |  destination = "worker-${count.index}.pem" |
|  |  |  } |
|  |  |  } |
|  |  |  |
|  |  |  resource "null\_resource" "tls\_kube\_proxy" { |
|  |  |  triggers = { |
|  |  |  tls\_ca = "${null\_resource.tls\_ca.id}" |
|  |  |  } |
|  |  |  |
|  |  |  provisioner "local-exec" { |
|  |  |  command = &lt;&lt;EOF |
|  |  |  cfssl gencert \ |
|  |  |  -ca=tls/ca.pem -ca-key=tls/ca-key.pem \ |
|  |  |  -config=tls/ca-config.json \ |
|  |  |  -profile=kubernetes \ |
|  |  |  tls/kube-proxy-csr.json \ |
|  |  |  \| cfssljson -bare tls/kube-proxy |
|  |  |  EOF |
|  |  |  } |
|  |  |  } |
|  |  |  |
|  |  |  resource "null\_resource" "tls\_kubernetes" { |
|  |  |  triggers = { |
|  |  |  tls\_ca = "${null\_resource.tls\_ca.id}" |
|  |  |  } |
|  |  |  |
|  |  |  provisioner "local-exec" { |
|  |  |  command = &lt;&lt;EOF |
|  |  |  cfssl gencert \ |
|  |  |  -ca=tls/ca.pem -ca-key=tls/ca-key.pem \ |
|  |  |  -config=tls/ca-config.json \ |
|  |  |  -hostname=10.32.0.1,${join\(",", aws\_instance.controller.\*.private\_ip\)},${join\(",", local.controller\_hostnames\)},${aws\_lb.k8s.dns\_name},127.0.0.1,kubernetes.default \ |
|  |  |  -profile=kubernetes \ |
|  |  |  tls/kubernetes-csr.json \ |
|  |  |  \| cfssljson -bare tls/kubernetes |
|  |  |  EOF |
|  |  |  } |
|  |  |  } |
|  |  |  |
|  |  |  resource "null\_resource" "tls\_controller" { |
|  |  |  triggers = { |
|  |  |  tls\_ca = "${null\_resource.tls\_ca.id}" |
|  |  |  tls\_kubernetes = "${null\_resource.tls\_kubernetes.id}" |
|  |  |  } |
|  |  |  |
|  |  |  \# Terraform does not allow to use \`length\` function on computed list |
|  |  |  \# count = "${length\(aws\_instance.controller.\*.id\)}" |
|  |  |  count = "${length\(var.controller\_ips\)}" |
|  |  |  |
|  |  |  connection { |
|  |  |  type = "ssh" |
|  |  |  user = "ubuntu" |
|  |  |  private\_key = "${tls\_private\_key.k8s.private\_key\_pem}" |
|  |  |  } |
|  |  |  |
|  |  |  provisioner "file" { |
|  |  |  source = "tls/ca.pem" |
|  |  |  destination = "/home/ubuntu/ca.pem" |
|  |  |  } |
|  |  |  |
|  |  |  provisioner "file" { |
|  |  |  source = "tls/ca-key.pem" |
|  |  |  destination = "/home/ubuntu/ca-key.pem" |
|  |  |  } |
|  |  |  |
|  |  |  provisioner "file" { |
|  |  |  source = "tls/kubernetes-key.pem" |
|  |  |  destination = "/home/ubuntu/kubernetes-key.pem" |
|  |  |  } |
|  |  |  |
|  |  |  provisioner "file" { |
|  |  |  source = "tls/kubernetes.pem" |
|  |  |  destination = "/home/ubuntu/kubernetes.pem" |
|  |  |  } |
|  |  |  } |


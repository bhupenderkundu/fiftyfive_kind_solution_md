#What symptoms you observed
1.  Invalid value for "path" parameter: no file exists at "./../kubernetes/cluster-config.yaml"; this function works only with files that are distributed as part of the configuration source code, so if this file
│ will be created by a resource in this configuration you must instead obtain this result from an attribute of that resource.
2. │ Error: Reference to undeclared input variable
│
│   on main.tf line 42, in resource "null_resource" "kind_cluster":
│   42:       kind create cluster --name ${var.cluster_name} --config ${var.kube_config_path}
│
│ An input variable with the name "kube_config_path" has not been declared. This variable can be declared with a variable "kube_config_path" {} block.
╵

#What tools you used to investigate

terraform plan run to get how terraform will create resource 

#What the root cause was and how you confirmed it

Invalid value for "path" parameter: no file exists at "./../kubernetes/cluster-config.yaml";

file name is not same as mentioned in code

variable name is not same as passed in main.tf " │   on main.tf line 42, in resource "null_resource" "kind_cluster":
│   42:       kind create cluster --name ${var.cluster_name} --config ${var.kube_config_path}
│
│ An input variable with the name "kube_config_path" has not been declared."



#What you did to fix it
Updated the file name and kube_config_path variable name as created with kind_config_path

#How you verified the fix

Terraform plan work smoothly
Terraform apply command works fine
null_resource.sabotage: Creation complete after 1m22s [id=5670533291352604169]

Apply complete! Resources: 4 added, 0 changed, 1 destroyed.

Outputs:

challenge_namespaces = tolist([
  "t2",
  "t3",
  "t5",
  "t6",
])
cluster_name = "sanjay-challenge"
cluster_nodes = {
  "control_plane" = "sanjay-challenge-control-plane"
  "workers" = [
    "sanjay-challenge-worker",
    "sanjay-challenge-worker2",
    "sanjay-challenge-worker3",
  ]
}

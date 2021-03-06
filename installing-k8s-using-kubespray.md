## Deploying k8s on AWS using KubeSpray

Recordings:
[Part1](https://www.youtube.com/watch?v=drzhR3LiCHM)
[Part2](https://www.youtube.com/watch?v=NHCaqdC8hBI)

1.  You need following

    - Ansible
    - Terraform
    - AWS Subscription

# Verify the required packages

1.  Login into your Linux machine or use Ubuntu bash on Windows
2.  Verify if ansible & terraform are installed

    ```
    $ which terraform
    $ which ansible
    ```

3.  To Install Terraform (Skip if installed!)

    ```
    $ sudo apt install unzip wget -y
    $ wget https://releases.hashicorp.com/terraform/0.14.5/terraform_0.14.5_linux_amd64.zip
    $ unzip terraform_0.14.5_linux_amd64.zip
    $ sudo cp terraform /usr/bin/
    $ cd /usr/bin
    $ sudo chmod +x terraform
    $ cd ~
    $ which terraform
    $ terraform -v
    ```

4.  To Install Ansible (Skip if installed!)

    ```
    $ sudo apt update -y
    $ sudo apt install ansible -y
    $ ansible --version
    ```    


5.  Clone the kubespray repository from GitHub

    ```
    $ sudo apt install git -y
    ### For LINUX VM
    $ cd ~
    ### FOR WSL/Ubuntu Bash
    $ cd /mnt/c/
    $ git clone https://github.com/kubernetes-sigs/kubespray
    ```

6.  Edit the file `contrib/terraform/terraform.tfvars` and make changes like this:

    ```ini
    #Global Vars
    aws_cluster_name = "mahendra-k8s"

    #VPC Vars
    aws_vpc_cidr_block       = "10.250.192.0/18"
    aws_cidr_subnets_private = ["10.250.192.0/20", "10.250.208.0/20"]
    aws_cidr_subnets_public  = ["10.250.224.0/20", "10.250.240.0/20"]

    #Bastion Host
    aws_bastion_size = "t2.micro"


    #Kubernetes Cluster
    aws_kube_master_num  = 1
    aws_kube_master_size = "t2.medium"

    aws_etcd_num  = 1
    aws_etcd_size = "t2.medium"

    aws_kube_worker_num  = 2
    aws_kube_worker_size = "t2.medium"

    #Settings AWS ELB

    aws_elb_api_port                = 6443
    k8s_secure_api_port             = 6443
    kube_insecure_apiserver_address = "0.0.0.0"

    default_tags = {
    Creator = "mahendra"
    #  Product = "kubernetes"
    }

    inventory_file = "../../../inventory/hosts"
    ```

7.  Create a credentials file `credentials.tfvars`

    ```ini
    #AWS Access Key
    AWS_ACCESS_KEY_ID = "[ACCESSKEY-ID]"
    #AWS Secret Key
    AWS_SECRET_ACCESS_KEY = "[ACCESS-KEY]"
    #EC2 SSH Key Name
    AWS_SSH_KEY_NAME = "k8s-pair"
    #AWS Region
    AWS_DEFAULT_REGION = "ap-south-1"
    ```

8.  Visit URL for creation of SSH-Key Pair (You will have to login)

9.  Enter UNIQUE name for SSHKeyPair and click "PEM" format and click `Create KeyPair`

10. Update your key-pair name in `credentials.tfvars` file

11. Initialize terraform deployment

    ```
    $ cd /mnt/c/kubespray/contrib/terraform/aws
    $ terraform init
    $ terraform plan -out=plan1 --var-file=credentials.tfvars
    ```

12. After successful deployment, a file would be created at `/mnt/c/kubespray/inventory/hosts`

13. Run following command from `/mnt/c/kubespray` location (Wait for 20/30 minutes)

    ```
    $ ansible-playbook -i inventory/hosts cluster.yml -e ansible_user=ubuntu -b --become_user=root
    ```

14. After Successful installation, we need to LOGIN inside the master-node. master-node, worker-node and etcd-node DO NOT HAVE public-ips!

15. To connect via Bastion-node, a new file "ssh-bastion.conf" is created in root directory "/mnt/c/kubespray" Open this file in text editor to verify the Public IP of Bastion node matches with public-ip of bastion node on AWS Console. (IP is mentioned in THREE places)

16. Now, using the SSH Key pair you created earlier to connect with the master node via bastion-node

    ```
    ## Lets assume ssh key is in downloads folders
    $ cd /mnt/c/Users/mahendra/Downloads
    $ cp k8s1.pem ~/.ssh/
    $ cd ~/.ssh
    # Set the permission on ssh-key
    $ chmod 0600 k8s1.pem
    $ eval $(ssh-agent)
    $ ssh-add -D
    $ ssh-add ~/.ssh/k8s1.pem
    $ ssh-add -L 
    $ cd /mnt/c/kubespray
    ### Replace 10.250.200.70 with Private IP of master-0 from AWS CONSOLE
    $ ssh -F ssh-bastion.conf ubuntu@10.250.200.70
    ## Copy the default credentials
    $ mkdir ~/.kube
    $ sudo cp /etc/kubernetes/admin.conf ~/.kube/config
    $ sudo chown ubuntu:ubuntu ~/.kube/config
    $ exit
    ```

17. Now, download the config from master node to local machine

    ```
    $ scp -F ssh-bastion.conf ubuntu@10.250.200.70:/home/ubuntu/.kube/config /mnt/c/Users/mahendra/.kube/config2
    ```

18. Open the downloaded config2 file with text-editor and replace the IP address of master node with DNS of elastic load balancer from aws-console.

19. Open a new command-prompt start using new cluster-config file

    ```
    $ set KUBECONFIG=c:\users\mahendra\.kube\config2
    $ kubectl get node
    ```
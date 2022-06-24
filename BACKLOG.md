# K3D CLUSTER
## Versions
### v0.0.1

* Install tools and create a basic cluster
### v0.0.2

* Instale o painel
* Expandir exemplos
### v0.0.3
* Instale o Prometheus-Graphana
### v0.0.4
* Criar arquivos de configuração
### v0.1.0
* Crie cluster usando arquivos de configuração em vez de parâmetros de linha de comando
* Instale o Nginx Ingress em vez do Traefik 1.6
* Use certificados de servidor
* Criar volumes persistentes
* Instale o Prometheus Graphana
## BACKLOG

* Configure o painel do k8s com o Ingress
* Instale o Rancher (https://gist.github.com/rafi/d4440661e7de208009701ca3627caa1d)
* Altere o Ingress para o Traefik v2 / Ingress
Por padrão, o k3s vem com o Traefik v1 como o controlador de ingresso padrão, na maioria das vezes eu prefiro trazer meu próprio controlador de ingresso, minha escolha pessoal é ingress-nginx porque é bastante simples e fácil de usar (e também uma brisa para configurar certificados TLS via cert-manager. (https://gist.github.com/rafi/d4440661e7de208009701ca3627caa1d)(https://royportas.com/posts/2020-11-20-setting-up-k3s-and-k3d/) (https ://codeburst.io/creating-a-local-development-kubernetes-cluster-with-k3s-and-traefik-proxy-7a5033cb1c2d)
* Mude Flanel para Calico (https://k3d.io/usage/guides/calico/)
* Instale certificados personalizados (https://sysadmins.co.za/https-using-letsencrypt-and-traefik-with-k3s/) (https://www.digitalocean.com/community/tutorials/how-to-set -up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes-es)
* Instale o EFK
* Crie um playbook ansible
* Criar clusters ha
* Melhore o registro k8s com k3d

* criar usuários e gerenciar funções de cluster (https://dev.to/martinsaporiti/instalando-k3d-para-jugar-con-k8s-phg)
* Gerenciar cluster com lente (https://github.com/lensapp/lens/releases/tag/v4.1.2)
* Implante o Istio (https://dev.to/bufferings/tried-k8s-istio-in-my-local-machine-with-k3d-52gg)
* Portainer (https://github.com/portainer/portainer-k8s)
kubectl apply -f https://raw.githubusercontent.com/portainer/portainer-k8s/master/portainer.yaml
https://load-balancer-ip:9000
* ...
* Kubeapps [em breve] [opcional]
* Istio [em breve] [opcional]
* ELK [em breve] [opcional]
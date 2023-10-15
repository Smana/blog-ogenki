+++
author = "Smaine Kahlouch"
title = "Sécuriser le Cloud avec `Tailscale` : Mise en œuvre d'un VPN simplifiée"
date = "2023-10-07"
summary = "Tailscale est une solution de **VPN** qui permet connecter des appareils ou serveurs de manière sécurisée. Comment en bénéficier pour accéder à une infrastruture Cloud?"
featured = true
codeMaxLines = 21
usePageBundles = true
toc = true
tags = [
    "security",
    "network"
]
thumbnail= "thumbnail.png"
+++

Lorsqu'on parle de sécurisation de l'accès aux ressources Cloud, l'une des règles d'or est d'éviter les expositions directes à Internet. La question qui se pose alors pour les Devs/Ops est : comment, par exemple, accéder à une base de données, un cluster Kubernetes ou un serveur via SSH sans compromettre la sécurité? </br>

Les **réseaux privés virtuels (VPN)** offrent une réponse en établissant un lien sécurisé entre différents éléments d'un réseau, indépendamment de leur localisation géographique. De nombreuses solutions existent, allant de modèles en SaaS aux solutions que l'on peut héberger soi-même, utilisant divers protocoles et étant soit open source, soit propriétaires.

Parmi ces options, je souhaitais vous parler de [**Tailscale**](https://tailscale.com/). Cette solution utilise le protocole `WireGuard`, réputé pour sa simplicité et sa performance.  Avec Tailscale, il est possible de connecter des appareils ou serveurs de manière sécurisée, comme s'ils étaient sur un même réseau local, bien qu'ils soient répartis à travers le monde.

## :bullseye: Nos objectifs

* Comprendre comment fonctionne `Tailscale`
* Mise en oeuvre d'une connexion sécurisée avec AWS en quelques minutes
* Interragir avec l'API d'un cluster EKS via un réseau privé
* Accéder à des services hébergés sur Kubernetes en utilisant le réseau privé

<center><img src="admin_console.png" width="400" /></center>

Pour le reste de cet article il faudra évidemment **créer un compte Tailscale**. A noter que l'authentification est déléguée à des fournisseurs d'identité tiers (ex: Okta, Onelogin, Google ...).

Lorsque le compte est crée, on a directement accès à la console de gestion ci-dessus. Elle permet notamment de lister les appareils connectés, de consulter les logs, de modifier la plupart des paramètres...

## 💡 Sous le capot

{{% notice info Terminologie %}}
**Mesh VPN**: Un _mesh VPN_ est un type de réseau VPN où chaque nœud (c'est-à-dire chaque appareil ou machine) est connecté à tous les autres nœuds du réseau, formant ainsi un maillage. À distinguer des configurations VPN traditionnelles qui sont conçues généralement "en étoile", où plusieurs clients se connectent à un serveur central.

**Zero trust**: Signifie que chaque demande d'accès à un réseau est traitée comme si elle venait d'une source non fiable. Une application ou utilisateur doit prouver son identité et être autorisée avant d'accéder à une ressource. On ne fait pas confiance simplement parce qu'une machine ou un utilisateur provient d'un réseau interne ou d'une certaine zone géographique.

**Tailnet**: Dès la première utilisation de Tailscale, un _Tailnet_ est crée pour vous et correspond à votre propre réseau privé. Chaque appareil dans un tailnet reçoit une IP Tailscale unique, permettant une communication directe entre eux. Chacun de ces réseaux possède son propre nom ainsi qu'un label associé à une organisation.
{{% /notice %}}

<center><img src="mesh.png" width="500" /></center>

L'architecture de Tailscale est conçue de telle sorte que le _Control plane_ et le _Data plane_ sont clairement séparés:

* D'une part, il y a le **serveur de coordination**. Son rôle est d'échanger des métadonnées et des clés publiques entre tous les participants d'un Tailnet (La clé privée étant gardée en toute sécurité son nœud d'origine).

* D'autre part, les nœuds du Tailnet s'organisent en un **réseau maillé** (_Mesh_). Au lieu de passer par le serveur de coordination pour échanger des données, ces nœuds communiquent directement les uns avec les autres en mode **point à point**. Chaque nœud dispose d'une identité unique pour s'authentifier et rejoindre le Tailnet.

## :inbox_tray: Installation du client

La majorité des plateformes sont supportées et les procédures d'installation sont listées [ici](https://tailscale.com/kb/installation/).
En ce qui me concerne je suis sur Archlinux:

```console
sudo pacman -S tailscale
```

Il est possible de démarrer le service automatiquement au démarrage de la machine.
```console
sudo systemctl enable --now tailscaled
```

Pour enregistrer son ordinateur perso, lancer la commande suivante:
```console
sudo tailscale up --accept-routes

To authenticate, visit:

        https://login.tailscale.com/a/f50...
```

<center><img src="laptop_connected.png" width="850" /></center>


ℹ️ l'option `--accept-routes` est nécessaire sur Linux et permettra d'accepter les routes annoncées par les Subnet routers. On verra cela dans la suite de l'article

Vérifier que vous avez bien obtenu une IP du réseau Tailscale:
```console
tailscale ip -4
100.118.83.67

tailscale status
100.118.83.67   ogenki               smainklh@    linux   -
```

ℹ️ Pour les utilisateurs de Linux, vérifier que Tailscale fonctionne bien avec votre configuration DNS: Suivre [cette documentation](https://tailscale.com/kb/1188/linux-dns/).

{{% notice tip "Les sources" %}}
Toutes les étapes réalisées dans cet article proviennent de ce [**dépôt git**](https://github.com/Smana/demo-secured-eks)

Il va permettre de créer l'ensemble des composants qui ont pour objectif d'obtenir un cluster EKS de Lab et font suite à un précédent article sur [Cilium et Gateway API](https://blog.ogenki.io/fr/post/cilium-gateway-api/).

{{% /notice %}}

## ☁️ Accéder aux réseaux privés sur AWS

![Subnet router](subnet_router.png)

Afin de pouvoir accéder de manière sécurisée à l'ensemble des ressources disponibles sur AWS, il est possible de déployer un _**Subnet router**_.

Un _Subnet router_ est une instance Tailscale qui permet d'accéder à des sous-réseaux qui ne sont pas directement liés à Tailscale. Il fait office de **pont** entre le réseau privé virtuel de Tailscale (_Tailnet_) et d'autres réseaux locaux.

Nous pouvons alors **router des sous réseaux du Clouder à travers le VPN de Tailscale**.

⚠️ Pour ce faire, sur AWS, il faudra bien entendu configurer les _security groups_ correctement pour autoriser les Subnet routers.


### 🚀 Déployer un Subnet router

Entrons dans le vif du sujet et deployons un _Subnet router_ sur un réseau AWS!</br>
Tout est fait en utilisant le code **Terraform** présent dans le répertoire [terraform/network](https://github.com/Smana/demo-secured-eks/tree/main/terraform/network). Nous allons analyser la configuration spécifique à Tailscale qui est présente dans le fichier [tailscale.tf](https://github.com/Smana/demo-secured-eks/blob/main/terraform/network/tailscale.tf) avant de procéder au déploiement.

#### Le provider Terraform

Il est possible de configurer certains paramètres au travers de l'**API** Tailscale grâce au [provider Terraform](https://github.com/tailscale/terraform-provider-tailscale).
Pour cela il faut au préalable génerer une clé d'API 🔑 sur la console d'admin:

<center><img src="api_key.png" width="750" /></center>

Il faudra conserver cette clé dans un endroit sécurisé car elle est utilisée pour déployer le Subnet router

```hcl
provider "tailscale" {
  api_key = var.tailscale.api_key
  tailnet = var.tailscale.tailnet
}
```

<ins>**les ACL's**</ins>

Les ACL's permettent de définir qui est autorisé à communiquer avec qui (utilisateur ou appareil). À la création d'un compte, celle-cis sont très permissives et il n'y a aucune restriction (tout le monde peut parler avec tout le monde).

```hcl
resource "tailscale_acl" "this" {
  acl = jsonencode({
    acls = [
      {
        action = "accept"
        src    = ["*"]
        dst    = ["*:*"]
      }
    ]
...
}
```
{{% notice note Note %}}
Pour mon environnement de Lab, j'ai conservé cette configuration par défault car je suis la seule personne à y accéder. De plus les seuls appareils connectés à mon Tailnet sont mon laptop et le Subnet router. En revanche dans un cadre d'entreprise, il faudra bien y réfléchir. Il est alors possible de définir une politique basée sur des groupes d'utilisitateurs ou sur les tags des noeuds.

Consulter cette [doc](https://tailscale.com/kb/1018/acls/) pour plus d'info.
{{% /notice %}}

<ins>**Les noms de domaines (DNS)**</ins>

Il y a [différentes façons](https://tailscale.com/kb/1054/dns/) possibles de gérer les noms de domaines avec Tailscale:

**Magic DNS**: Lorsqu'un appareil rejoint le Tailnet, il s'enregistre avec un nom et celui-ci peut-être utilisé directement pour communiquer avec l'appareil.
```console
tailscale status
100.118.83.67   ogenki               smainklh@    linux   -
100.115.31.152  ip-10-0-43-98        smainklh@    linux   active; relay "par", tx 3044 rx 2588

ping ip-10-0-43-98
PING ip-10-0-43-98.tail9c382.ts.net (100.115.31.152) 56(84) bytes of data.
64 bytes from ip-10-0-43-98.tail9c382.ts.net (100.115.31.152): icmp_seq=1 ttl=64 time=11.4 ms
```

**AWS**: Pour utiliser les noms de domaines internes à AWS il est possible d'utiliser la [deuxième IP du VPC](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.html#AmazonDNS) qui correspond toujours au serveur DNS. Cela permet d'utiliser les éventuelles zones privées sur route53 ou de se connecter aux ressources en utilisant les noms de domaines.

La configuration la plus simple est donc de déclarer la liste des serveurs DNS à utiliser et d'y ajouter celui de AWS. Ici un exemple avec le DNS publique de Cloudflare.

```hcl
resource "tailscale_dns_nameservers" "this" {
  nameservers = [
    "1.1.1.1",
    cidrhost(module.vpc.vpc_cidr_block, 2)
  ]
}
```

<ins>**La clé d'authentification ("auth key")**</ins>

Pour qu'un appareil puisse rejoindre le Tailnet au démarrage il faut que Tailscale soit démarré en utilisant une clé d'authentification. Celle-ci est générée comme suit

```hcl
resource "tailscale_tailnet_key" "this" {
  reusable      = true
  ephemeral     = false
  preauthorized = true
}
```

* `reusable`: S'agissant d'un `autoscaling group`, il faut que cette même clé puisse être utilisée plusieurs fois.
* `ephemeral`: Pour cette démo nous créons une clé qui n'expire pas. En production il serait préférable d'activer l'expiration.
* `preauthorized`: Il faut que cette clé soit déjà valide et autorisée pour que l'instance rejoigne automatiquement le Tailscale.

La clé ainsi générée est utilisée pour lancer tailscale avec le paramètre `--auth-key`
```console
sudo tailscale up --authkey=<REDACTED>
```


<ins>**Annoncer les routes pour les réseaux AWS**</ins>

Enfin il faut annoncer le réseau que l'on souhaite faire passer par le _Subnet router_. Dans notre exemple, nous décidons de router tout le réseau du VPC qui a pour CIDR `10.0.0.0/16`.

Afin que cela soit possible de façon automatique, il y a une règle [autoApprovers](https://tailscale.com/kb/1018/acls/#auto-approvers-for-routes-and-exit-nodes) à ajouter. Cela permet d'indiquer que les routes annoncées par l'utilisateur `smainklh@gmail.com` sont autorisées sans que cela requiert une étape d'approbation.

```hcl
    autoApprovers = {
      routes = {
        "10.0.0.0/16" = ["smainklh@gmail.com"]
      }
    }
```

La commande lancée au démarrage de l'instance _Subnet router_ est la suivante:
```console
sudo tailscale up --authkey=<REDACTED> --advertise-routes="10.0.0.0/16"
```

#### Le module Terraform

J'ai créé un [module](https://github.com/Smana/terraform-aws-tailscale-subnet-router) très simple qui permet de déployer un `autoscaling group` sur AWS et de configurer Tailscale. Au démarrage de l'instance, elle s'authentifiera en utilisant une `auth_key` et annoncera les réseaux indiqués. Dans l'exemple ci-dessous l'instance annonce le CIDR du VPC sur AWS.

```hcl
module "tailscale_subnet_router" {
  source  = "Smana/tailscale-subnet-router/aws"
  version = "1.0.4"

  region = var.region
  env    = var.env

  name     = var.tailscale.subnet_router_name
  auth_key = tailscale_tailnet_key.this.key

  vpc_id                = module.vpc.vpc_id
  subnet_ids            = module.vpc.private_subnets
  advertise_routes      = [module.vpc.vpc_cidr_block]
...
}
```

Maintenant que nous avons analysé les différents paramètres, il est temps de **démarrer notre Subnet router** 🚀 !! </br>

Il faut au préalable créer un fichier `variable.tfvars` dans le répertoire [terraform/network](https://github.com/Smana/demo-secured-eks/tree/main/terraform/network).

```hcl
env                 = "dev"
region              = "eu-west-3"
private_domain_name = "priv.cloud.ogenki.io"

tailscale = {
  subnet_router_name = "ogenki"
  tailnet            = "smainklh@gmail.com"
  api_key            = "tskey-api-..."
}

tags = {
  project = "demo-secured-eks"
  owner   = "Smana"
}
```


Puis lancer la commande suivante:
```console
tofu plan --var-file variables.tfvars
```

Après vérification du plan, appliquer les changements

```console
tofu apply --var-file variables.tfvars
```

Quand l'instance est démarrée, elle apparaitra dans la liste des appareils du Tailnet.
```console
tailscale status
100.118.83.67   ogenki               smainklh@    linux   -
100.68.109.138  ip-10-0-26-99        smainklh@    linux   active; relay "par", tx 33868 rx 32292
```

Nous pouvons aussi vérifier que la route est bien annoncée comme suit:
```console
tailscale status --json|jq '.Peer[] | select(.HostName == "ip-10-0-26-99") .PrimaryRoutes'
[
  "10.0.0.0/16"
]
```

⚠️ Pour des raisons de sécurité, pensez à supprimer le fichier `variables.tfvars` car il contient la clé d'API.

👏 Et voilà ! Nous sommes maintenant en mesure d'**accéder au réseau sur AWS**, à condition d'avoir également configuré les règles de filtrage, comme les ACL et les security groups. Nous pouvons par exemple accéder à une base de données depuis le poste de travail

```console
psql -h demo-tailscale.cymnaynfchjt.eu-west-3.rds.amazonaws.com -U postgres
Password for user postgres:
psql (15.4, server 15.3)
SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, compression: off)
Type "help" for help.

postgres=>
```

### 💻 Une autre façon de faire du SSH

Traditionnellement, nous devons parfois nous connecter à des serveurs en utilisant le protocole SSH. Pour ce faire, il faut générer une clé privée et distribuer la clé publique correspondante sur les serveurs distants.

Contrairement à l'utilisation des clés SSH classiques, étant donné que Tailscale utilise `Wireguard` pour l'authentification et le chiffrement des connexions il n'est **pas nécessaire de ré-authentifier le client**. De plus, Tailscale gère également la distribution des clés SSH d'hôtes. Les règles ACL permettent de révoquer l'accès des utilisateurs sans avoir à supprimer les clés SSH. De plus, il est possible d'activer un mode de vérification qui renforce la sécurité en exigeant une ré-authentification périodique. On peut donc affirmer que l'utilisation de `Tailscale SSH` **simplifie** l'authentification, la gestion des connexions SSH et **améliore le niveau de sécurité**.

ℹ️ Avec Tailscale SSH il est possible de se connecter en SSH peu importe où est situé l'appareil. En revanche dans un contexte 100% AWS, on préferera probablement utiliser [AWS SSM](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html).

Les autorisations pour utiliser SSH sont aussi gérées au niveau des ACL's
```hcl
...
    ssh = [
      {
        action = "check"
        src    = ["autogroup:member"]
        dst    = ["autogroup:self"]
        users  = ["autogroup:nonroot"]
      }
    ]
...
```

La règle ci-dessus autorise tous les utilisateurs à accéder à leurs propres appareils en utilisant SSH. Lorsqu'ils essaient de se connecter, ils doivent utiliser un compte utilisateur autre que `root`.
Pour chaque tentative de connexion, une authentification supplémentaire est nécessaire (`action=check`). Cette authentification se fait en visitant un lien web spécifique

```console
ssh ubuntu@ip-10-0-26-99
...
# Tailscale SSH requires an additional check.
# To authenticate, visit: https://login.tailscale.com/a/f1f09a548cc6
...
ubuntu@ip-10-0-26-99:~$
```

Les logs d'accès à la machine peuvent être consultés en utilisant `journalctl`
```console
ubuntu@ip-10-0-26-99:~$ journalctl -aeu tailscaled|grep ssh
Oct 15 15:51:34 ip-10-0-26-99 tailscaled[1768]: ssh-conn-20231015T155130-00ede660b8: handling conn: 100.118.83.67:55098->ubuntu@100.68.109.138:22
Oct 15 15:51:56 ip-10-0-26-99 tailscaled[1768]: ssh-conn-20231015T155156-b6d1dc28c0: handling conn: 100.118.83.67:44560->ubuntu@100.68.109.138:22
Oct 15 15:52:52 ip-10-0-26-99 tailscaled[1768]: ssh-conn-20231015T155156-b6d1dc28c0: starting session: sess-20231015T155252-5b2acc170e
Oct 15 15:52:52 ip-10-0-26-99 tailscaled[1768]: ssh-session(sess-20231015T155252-5b2acc170e): handling new SSH connection from smainklh@gmail.com (100.118.83.67) to ssh-user "ubuntu"
Oct 15 15:52:52 ip-10-0-26-99 tailscaled[1768]: ssh-session(sess-20231015T155252-5b2acc170e): access granted to smainklh@gmail.com as ssh-user "ubuntu"
Oct 15 15:52:52 ip-10-0-26-99 tailscaled[1768]: ssh-session(sess-20231015T155252-5b2acc170e): starting pty command: [/usr/sbin/tailscaled be-child ssh --uid=1000 --gid=1000 --groups=1000,4,20,24,25,27,29,30,44,46,115,116 --local-user=ubuntu --remote-user=smainklh@gmail.com --remote-ip=100.118.83.67 --has-tty=true --tty-name=pts/0 --shell --login-cmd=/usr/bin/login --cmd=/bin/bash -- -l]
```

### ☸ Plusieurs options sur Kubernetes

* Subnet router
* Proxy
* Sidecar
* Operator

```console
  CURRENT_K8S_URL=$(kubectl config view --minify --output=jsonpath="{.contexts[?(@.name=='$(kubectl config current-context)')].context.cluster}")
  dig +short ${CURRENT_K8S_URL}
```

```hcl
module "eks" {
...
  cluster_security_group_additional_rules = {
    ingress_source_security_group_id = {
      description              = "Ingress from the Tailscale security group to the API server"
      protocol                 = "tcp"
      from_port                = 443
      to_port                  = 443
      type                     = "ingress"
      source_security_group_id = data.aws_security_group.tailscale.id
    }
  }
...
}
```


## 👐 Open source et tarifs

Pricing raisonnable avec une vrai politique OpenSource et je suis pour payer une solution lorsqu'elle le mérite et que cela reste raisonnable. Encourager ces sociétés qui ont une sensibilité Open Source

Le [client Tailscale](https://github.com/tailscale/tailscale) est Open source sous license `BSD 3-Clause`

Self hosted <https://github.com/juanfont/headscale> non lié à la société Tailscale.

## 💭 Dernières remarques

Cloudflare, interface web, latencie. Ça fait longtemps.

Ce qui distingue Tailscale des autres VPN, c'est sa capacité à configurer des connexions de manière quasi instantanée sans nécessiter de configuration complexe.

## 🔖 References

<https://tailscale.com/blog/how-tailscale-works/#the-control-plane-key-exchange-and-coordination>
<https://tailscale.com/blog/2019-09-10-zero-trust/>


<https://cert-manager.io/docs/configuration/ca/>

```console
sudo tailscale up --force-reauth --accept-routes
```

#   Projet Dématerialisation rh-entretien-pro
#   Formulaire avec Orbeon Form


## 1 Installation et configuration du serveur :
## Nom du server : rh-apps2019.univ.run
## ip : 10.82.70.195

Les outils nécessaires : \
    - java\
    - apache2\
    - apache tomcat8\
    - Orbeon PE (avec licence)\
    - Postgresql (version 9.4 et plus; j'ai utilisé la version 11)\

Verifier si java est installé 
```
java --version
```

Sinon, faire une installation du Kit de developpement JAVA
```
sudo apt-get update
sudo apt install default-jdk
sudo apt install default-jre
```
## 1.1 Téléchargement des outils
### 1.1.1 Serveur web Apache
Installer apache2 comme serveur web 
```
sudo apt install apache2
```
### 1.1.2 Apache Tomcat
Pour deployer Orbeon, on a besoin du serveur Tomcat,
la version courante utilisé est la version 8

```
sudo apt install tomcat8
```
 - Facultatif :
si vous avez besoin de la page "manager" :
```
sudo apt install tomcat8-admin
```
 - Configuration de Tomcat8
Pour avoir un droit d'administrateur aller dans /var/lib/tomcat8/conf/ (dossier d'installation par défaut) puis editer le fichier tomcat-users.xml :
```
  <user username="nom d'utilisateur" password="votre mot de passe" roles="manager-gui,admin-gui"/>

```
Les roles "manger-gui et admin-gui" permet d'avoir le droit d'administrer Tomcat

 - Librairie :
Pour tous les applications qu'on va deployer dans Tomcat, veuillez bien vous renseignez sur la version des librairies qui sont compatibles avec la version du Tomcat installé (8.0.35)\
Tous les librairies doivent être copié dans /var/lib/tomcat8/lib/ , après il faut recharger Tomcat

 - Ressources :
En effet, nos applications auront besoin d'éventuelle ressource, comme l'interaction avec une base de donnée, avec LDAP, ou avec le CAS. C'est mieux si on les configures dans le contexte de chaque application (META-INF/context.xml ou WEB-INF/web.xml) au lieu de les chargés dans Tomcat (server.xml; context.xml) 

### 1.1.3 Orbeon Form
Orbeon existe sous 2 variantes : CE et PE (payante), on va utilisé la version PE car elle offre plusieurs fonctinnalité indispensable pour nos attente comme un système d'authentification, ... \
La licence est de 1000$ par ans. \
Aller dans https://www.orbeon.com , télécharger Orbeon PE, et n'oublier pas de récupérer la licence (3 mois d'essai). \
Cette licence d'essai gratuit peut être renouvellé tous les 3 mois.\
Après la téléchargement, vous aurez un fichier orbeonPE.zip, faite une extraction puis copier le fichier orbeon.zip dans le dossier webapp de Tomcat.

```
sudo apt install unzip
sudo unzip orbeonPE.zip
cd orbeonPE/
cp orbeon.war /var/lib/tomcat8/webapps/
```

#### La version actuelle d'orbeon est : 2018.2
Renommer orbeon.war par le nom de l'application : entretienpro.war dans notre cas.\
Quand orbeon est deployé, ne le lance pas tout de suite car il faut copier la licence. Aller dans le repertoire où se trouve la licence (licence.xml) puis copier la dans :
```
/var/lib/tomcat8/webapps/entretienpro/WEB-INF/ressource/config/
```
## 2 - Configuration d'Orbeon Form (back end)

### 2.1 Système d'authentification

### 2.1.1 Configuration de LDAP dans Tomcat

En terme de fonctinnalité, le client veut avoir son formulaire identifier par son propre compte, peut être modifier par lui même ou par son administrateur.\
On a donc besoin de bien définir un role d'administrateur (CRUD : Create Read Update Delete), et le role d'un client avec un droit limiter.

On va donc utiliser l'annuaire LDAP de l'université, qui fourni le nom d'utilisateur, un rôle bien précis, et qu'on peut créer un groupe pour filter les utilisateurs.

L'application rh-entretien-pro s'addresse aux employés, ce sont les responsables qui completent le formulaire, donc on utilise le filtre : ((&amp;(supannAliasLogin={0})(eduPersonAffiliation=employee)).

On a créer un groupe cn=rh-entretien-pro, dont les membres ont le droit d'administrateurs dans l'application orbeon.

Mettre ensuite la ressource suivante dans :
```
/var/lib/tomcat8/webapps/entretienpro/META-INF/context.xml
```
La raison pour laquelle on a pas mis cette resource dans server.xml de tomcat est que le filtre est spécifique à l'application rh-entretien-pro.  

```
<Realm className="org.apache.catalina.realm.JNDIRealm"
               connectionURL="ldap://ldapr.univ.run:389"
               userBase="ou=People,dc=univ-reunion,dc=fr"
               userSearch="((&amp;(supannAliasLogin={0})(eduPersonAffiliation=employee))"
               roleBase="cn=rh-entretien-pro,ou=Groups,dc=univ-reunion,dc=fr"
               roleName="cn"
               roleSearch="(member={0})" />
```
On a donc une ressource pret à utilisé, maintenant il faut configurer orbeon pour integrer ces éléments.

### 2.1.2 Configuration de LDAP dans Orbeon

Comme système d'authentification, orbeon utilise la methode basé sur le conteneur (tomcat dans notre cas).
Tous les configurations d'integration d'orbeon s'effectuent dans le fichier:

```
/var/lib/tomcat8/webapps/entretienpro/WEB-INF/ressource/config/properties-local.xml
```
La classe "oxf.fr.authentication.method" recupère les informations d'identification de "value=container", c'est-à-dire de Tomcat.

"oxf.fr.authentication.container.roles" indique les rôles qui sont autorisés à acceder à l'application. La valeur '*' indique que tous les utilisateurs sont autorisés.


```
<properties>
<!-- Config ldap authentication -->
 
     <property as="xs:boolean"
	          name="oxf.fr.authentication.user-menu.enable"
       		  value="true"/>
    
    <property as="xs:string"
       		  name="oxf.fr.authentication.method"
       		  value="container"/>

	<property as="xs:string"
		  name="oxf.fr.authentication.container.roles"
		  value="rh-entretien-pro *" />  

</properties>
```

Aller ensuite dans web.xml de l'application entretienpro:


```
/var/lib/tomcat8/webapps/entretienpro/WEB-INF/web.xml
```
Puis ajouter les lignes suivantes:

```
<security-constraint>
        <web-resource-collection>
            <web-resource-name>Form Runner</web-resource-name>
            <url-pattern>/fr/auth</url-pattern>
        </web-resource-collection>
        <auth-constraint>
            <role-name>*</role-name>
        </auth-constraint>
    </security-constraint>

    <security-constraint>
        <web-resource-collection>
            <web-resource-name>Form Runner</web-resource-name>
            <url-pattern>/fr/*</url-pattern>
        </web-resource-collection>
        <auth-constraint>
            <role-name>*</role-name>
        </auth-constraint>
    </security-constraint>

```
Les lignes ci-dessus sont la configuration de l'autorisation à l'application. \
La balise  "url-pattern" indique le ressource à appliqué le droit d'autorisation, ici on a "/fr/*", qui veut dire tous les ressources après "/fr/" \

La balise "auth-constrains" indique les rôles autorisés à acceder au ressource. Ici on a tout autorisé (role-name : *).\
On a tout autorisé car déjà, le LDAP filtre les utilisateurs, tous les responsables peu importe leur rôle ont le droit de remplir le formulaire, et seul ceux qui sont membres du groupe rh-entretien-pro sont les administrateurs de l'application, donc pas besoin de mettre un super filtre ici.

Aller ensuite dans :

```
/var/lib/tomcat8/webapps/entretienpro/WEB-INF/ressource/config/form-builder-permissions.xml 
```
Ce fichier assigne à un utilisateur le droit d'administrateur.
Ici on va assigné a tous les utilisateurs qui ont pour rôle "rh-entretien-pro" le droit admin à tous les formulaires de l'application, ainsi ils priilégient le droit de créér ou de faire une modification dans form builder. 

```
<roles>
    <role name="rh-ebtretien-pro" app="*" form="*"/>
</roles>
```

## 2.2 Configuration de la gestion de la base de donnée d'Orbeon

Par défaut, les données d'orbeon sont stockés dans une base de données eXist-db, intégré avec l'apppication elle même. 

Il est necessaire de pointer les données d'orbeon dans un autre SGBD, puisque eXist-db ne supporte pas tous les fonctionnalités d'orbeon,
difficile à administrer.

### 2.2.1 Installation de postgresql
On utilise la version 11 de postgresql

```
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

echo "deb http://apt.postgresql.org/pub/repos/apt/ ${RELEASE}"-pgdg main | sudo tee  /etc/apt/sources.list.d/pgdg.list

sudo apt update
sudo apt install postgresql-11
```

### 2.2.2 Configuration de postgresql

On a demandé l'ouverture du port 5432 entre le serveur prod rh-apps2019.univ.run (10.82.70.195) et les machines de la DSI (10.75.x.x.).

#### a - Configurer l'accès externe de postgres

Par défaut postgres n'accepte que la connexion en local donc il faut modifier par l'editeur de votre choix les fichiers suivantes

```
/etc/postgresql/11/main/pg_hba.conf

# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    all             all             0.0.0.0/0               md5

```
cette ligne permet à tous les utilisateurs externes de se connecter à tous les bases de données, et sont sécurisés par md5. Cependant, vous pouvez spécifié les utilisateurs et la base de donnée.

Modifiez ensuite dans la partie CONNECTION AND AUTHENTICATION du fichier posgresql.conf:

```
/etc/postgresql/11/main/postgresql.conf

listen_addresses = '*'
```
cette ligne permet d'écouter tous les addresses demandant le port 5432 au lieu de localhost seulement.

Verification : 

```
ss -4tunelp | grep 5432
tcp    LISTEN     0      128       *:5432                  *:*                   users:(("postgres",pid=699,fd=3)) uid:119 ino:15958 sk:23 <->
```
on peut bien voir le '*:5432', qui confirme l'accès exterieur.

#### b - Création de la base de données

Se conneter en tant qu'utilisateur postgres
```
su - postgres
```
donnée un mot de passe à l'utilisateur postgres
```
psql -c "alter user postgres with password 'motdepasse'"
```
Créer une base de donnée, et donnée à l'utilisateur postgres les droits nécessaires

```
psql

CREATE DATABASE orbeon;
grant all privileges on database orbeon to postgres;
```

Maintenant vous pouvez accedé à la base :
```
url="jdbc:postgresql://localhost:5432/postgres"
username="postgres" 
password="motdepasse"
```
#### c - Création des tables 

Pour l'integration des SGBD avec orbeon, orbeon fourni un DLL pour chaque base en tenant compte de la version d'orbeon utilisé, télécharger le suivant le lien suivant. \
https://github.com/orbeon/orbeon-forms/blob/master/form-runner/jvm/src/main/resources/apps/fr/persistence/relational/ddl/postgresql-2018_2.sql

Connectez vous via une interface de gestion ou d'administration de base de données (pgadmin4, netbeans, ...), puis executer les requetes dans la base de donnée 'orbeon' créer ci-dessus.

#### Par la suite, les données d'orbeon sont dans la table : orbeon_form_data

##  3 - Configuration d'Orbeon Form (Font end)

<img src = "https://www.google.fr/images/srpr/logo11w.png" title = "google logo" alt = "Google logo">
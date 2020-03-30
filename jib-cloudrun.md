# Le mariage de Jib et Cloud Run

Le point de non retour a été atteint et on ne peut plus faire marche arrière, les containers font parti intégrantes du paysage de notre métier. 
### Mais que dois-je faire ?! 
- c’est du travail d’ops, f*** off !!! 
- Docker, pourquoi pas, mais je fais tourner ça comment ??? 
- Kubernetes ? Bordel j’ai déjà du mal à le dire 😱, alors comprendre le concept, j’en ai pour des mois 
- Google n'aurait pas fait un truc pour me faciliter la vie ?
 Effectivement, et comme souvent, Google nous facilite la vie et pas seulement avec une barre de recherche.
## JIB
### Késako ???
Jib est un outil créé par Google qui permet de générer des containers sans le Docker Deamon sur votre machine ni Dockerfile dans votre projet, mais ce n’est pas tout. Généralement, une application Java est défini par un seul layer de type jar qui regroupe l’ensemble du code et des dépendances. Jib propose de découper ce jar en plusieurs layers (classes et dépendences) et permettre un build d’image plus rapide et plus optimisé afin de ne recréer que les layers qui ont été modifiés. Ces layers sont ensuite ajoutés à une base image Java "distroless", qui ne contient exclusivement que les dépendances nécessaires au fonctionnement d’une JVM.
Pour plus d’information => [Distroless](https://github.com/GoogleContainerTools/distroless)
### KomanFège ??
Après toutes ces belles promesses, comment fait-on pour utiliser tout ça ?  Il existe des plugins pour Maven et pour Gradle. Et si vous êtes un peu maso (#sbt), vous pouvez l’implémenter vous même avec la lib jib-core. Nous verrons ici le fontionnement avec le plugin Maven.

## Let's code

On se place ici dans le contexte d'une application Spring Boot générée depuis ‎Spring Initializr.

Commençons par la configuration Maven pour le plugin Jib

```xml
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>2.1.0</version>
    <configuration>
        <to>
            <image>charon11/jib-demo</image>
            <tags>${project.version}</tags>
        </to>
    </configuration>
</plugin>
```

et ... ben c'est fini, il n'y a rien d'autre à faire, tout est prêt pour créer notre container.

Dans notre cas, nous allons build le container sans Docker.  
```bash
mvn jib:build
```

Que va t'il se passer ici:
- téléchargement de l'image distroless java sur gcr.io
- création du container à partir de l'image distroless
- upload du container sur la registry saisie dans la configuration (ici Docker Hub). Il faut donc d'abord être connecté sur Docker Hub à l'aide la commande "docker login". Il est possible de préciser les credentials dans la configuration maven.
- done

Il est également possible de builder son container avec Docker, à l'aide de la commande 
```bash
mvn jib:dockerBuild
```
Votre container est du coup upload dans la registry locale.
 
 Oui mais maintenant, il est temps de faire tourner tout ça.

 ## Cloud Run

 ### Re késako ?
 Cloud Run est une plate-forme de service totalement managé de Google permétant de déployer et d'exécuter des containers stateless. Le but de cette plateforme est de profiter des avantages des containers mais sans les potentielles difficultés du management d'une plateforme d'orchestration de container. Autre avantage, la facturation ne se fait que lorsque le container est en activité. On peut assimiler Cloud Run à une Cloud Function mais avec un container. 

### Let's use it

Si on prend en compte que vous avez déjà un projet dans la GCP, vous pouvez accédez au service Cloud Run via la barre de recherche de la console Google Cloud Platform

![Find Cloud Run](https://raw.githubusercontent.com/Charon11/jib-cloud-run/master/resources/GotoCloudRun.png)

Il faut ensuite activé le service Cloud Run: 

![Activate Cloud Run](https://raw.githubusercontent.com/Charon11/jib-cloud-run/master/resources/CloudRunActivateService.png)


![Cloud Run Service](https://raw.githubusercontent.com/Charon11/jib-cloud-run/master/resources/CloudRunService.png)

Vous allez ensuite devoir activer Api Registry:

![Cloud Run Registry](https://raw.githubusercontent.com/Charon11/jib-cloud-run/master/resources/CloudRunActivateRegistry.png)


Et enfin pour terminer, il faut installer le Google Cloud SDK en suivant la documentation suivante [Google Cloud SDK](https://cloud.google.com/sdk/docs), puis configurer [gcloud comme Docker credential helper](https://cloud.google.com/container-registry/docs/advanced-authentication#gcloud-helper)


Une fois que tout est installé et configuré, on est up and ready pour utiliser Cloud Run

Nous allons commencer par faire une petite modification dans la configuration de Jib :

```xml
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>2.1.0</version>
    <configuration>
        <to>
            <image>gcr.io/cloud-run-jib/jib-demo</image>
            <credHelper>gcloud</credHelper>
            <tags>${project.version}</tags>
        </to>
    </configuration>
</plugin>
```

Maintenant si vous relancez la commande
```bash
mvn jib:build
...
[INFO] Built and pushed image as gcr.io/cloud-run-jib/jib-demo, gcr.io/cloud-run-jib/jib-demo:0.0.1-SNAPSHOT
...
```
vous devriez build et push votre image sur la registry GCloud.

![Cloud Run Registry Container](https://raw.githubusercontent.com/Charon11/jib-cloud-run/master/resources/CloudRunRegistryContainer.png)

![Cloud Run Registry Container Version](https://raw.githubusercontent.com/Charon11/jib-cloud-run/master/resources/CloudRunRegistryContainerVersion.png)








Et enfin il va falloir activer Cloud Build


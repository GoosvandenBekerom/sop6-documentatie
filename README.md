# SOP6 Aanpak
*Goos van den Bekerom & Marvin Zwolsman*

## Introductie
Voor het vak SOP6 gaan we een geautomatiseerde ontwikkelstraat opbouwen. In dit document gaan we bijhouden welke stappen we hiervoor hebben genomen. We gaan hiervoor zo min mogelijk tekst schrijven en zoveel mogelijk laten zien wat we gedaan hebben. Denk aan uitgevoerde command line commando's, stukjes code en screenshots.

## Jenkins
#### Installatie
We gaan beginnen met het inrichten van een Jenkins docker container en het installeren van een pipeline hierop.
1. `$ docker pull jenkins/jenkins:alpine`
2. `$ docker run -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home jenkins/jenkins:alpine`

Doormiddel van de `-v` flag is er lokaal een docker volume aangemaakt ganaamd `jenkins_home` voor de te installeren plugins e.d.

Vervolgens zijn we via `http://localhost:8080` Jenkins als volgt gaan installeren
> Installatie optie: Install suggested plugins

Tot slot hebben we een tweetal plugins geïnstalleerd, de de user interface van Jenkins wat gebruiksvriendelijker maken.
- Locale - zodat het taalgebruik binnen Jenkins overal gelijk is.
- Blue Ocean - Voor het mooier weergeven van Pipelines

#### Inrichting pipeline
We gaan een Jenkins pipeline maken voor het automatisch builden, testen en deployen van het microservices onderdeel uit onze proftaak. Dit project is te vinden op [Github](https://github.com/GoosvandenBekerom/er-microservices).

We hebben binnen Jenkins een pipeline aangemaakt die gelinkt is aan het github project (Omdat deze container momenteel lokaal draait werkt een Github hook nog niet). Vervolgens hebben we een `Jenkinsfile` aangemaakt binnen het project. Dit bestand beschrijft de stappen die de pipeline moet doorlopen. Hieronder een simpele versie van dit bestand.

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'chmod +x ./gradlew && ./gradlew clean build'
                archiveArtifacts artifacts: '**/build/libs/*.war', fingerprint: true
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying....'
            }
        }
    }
}
```

## Artifactory
#### Installatie
Na het builden van de war files in het project willen we dat Jenkins deze deployed naar een soort repository. Hiervoor gaan we artifactory gebruiken.
We hebben de volgende commando's uitgevoerd om de artifactory container te starten:
1. `$ docker pull docker.bintray.io/jfrog/artifactory-oss:latest`
2. `$ docker volume create --name artifactory5_data`
3. `$ docker run --name artifactory-oss -d -v artifactory5_data:/var/opt/jfrog/artifactory -p 8081:8081 docker.bintray.io/jfrog/artifactory-oss:latest`

## Netwerk
#### Inrichten
Om Jenkins te kunnen laten deployen naar artifactory, hebben we een docker netwerk aangemaakt. Dit hebben we doormiddel van de volgende commando's gedaan:
1. `$ docker network create sop6`
2. `$ docker network connect sop6 artifactory-oss`
3. `$ docker network connect sop6 3a7a2d0a8fe7` (Dit is jenkins, we zijn vergeten hieraan een naam te geven)

Vervolgens hebben we de `Artifactory` plugin geïnstalleerd in Jenkins. In onderstaande screenshot is te zien hoe we deze plugin hebben ingericht.
![Screenshot](https://cdn.discordapp.com/attachments/380439326950948874/423465080043208705/unknown.png)

#### Wijzigingen Pipeline
Omdat Jenkins nu onze artifacts moet doorsturen naar de artifactory repository, hebben we de volgende aanpassingen gedaan in de Jenkinsfile.
```groovy
node {
    def server
    def buildInfo
    def rtGradle

    stage ('Clone') {
        checkout scm
        sh 'chmod +x ./gradlew'
    }

    stage ('Artifactory configuration') {
        server = Artifactory.server 'arti'

        rtGradle = Artifactory.newGradleBuild()
        rtGradle.useWrapper = true
        rtGradle.deployer repo: 'sop6-local', server: server
        rtGradle.resolver repo: 'sop6-virt', server: server
        rtGradle.deployer.deployArtifacts = false

        buildInfo = Artifactory.newBuildInfo()
    }

    stage ('Tests') {
        rtGradle.run rootDir: './', buildFile: 'build.gradle', tasks: 'clean test'
    }

    stage ('Deploy') {
        rtGradle.run rootDir: './', buildFile: 'build.gradle', tasks: 'artifactoryPublish', buildInfo: buildInfo
        rtGradle.deployer.deployArtifacts buildInfo
    }

    stage ('Publish build info') {
        server.publishBuildInfo buildInfo
    }
}
```

In onderstaande screenshots is te zien dat de Jenkins pipeline succesvol verlopen is en dat de war files in Artifactory staan.
![Screenshot artifactory 1](https://cdn.discordapp.com/attachments/380439326950948874/423468035215458315/unknown.png)
![Screenshot artifactory 2](https://cdn.discordapp.com/attachments/380439326950948874/423468003552657428/unknown.png)

## SonarQube
Om de kwaliteit van onze code automatisch te kunnen testen hebben we SonarQube toegevoegd aan onze automatische ontwikkelstraat. Dit hebben we gedaan doormiddel van de volgende commando's
1. `$ docker pull sonarqube:7.0-alpine`
2. `$ docker run -d --name sonarqube -p 9000:9000 -p 9092:9092 sonarqube:7.0-alpine`
3. `$ docker network connect sop6 sonarqube`

Nu SonarQube draait binnen ons docker netwerk hebben we dit moeten configureren in ons project. Dit hebben we gedaan door deze plugin toe te voegen aan het `build.gradle` bestand:
```groovy
plugins {
    id "org.sonarqube" version "2.6"
}
```
Daarna hebben we deze stage toegevoegd aan de `Jenkinsfile`
```groovy
stage('SonarQube analysis') {
    withSonarQubeEnv('er-microservices') {
        sh './gradlew --info sonarqube'
    }
}
```
In deze screenshot is te zien dat SonarQube nu draait en dat het de code heeft geanalyseerd.
![Screenshot SonarQube](https://cdn.discordapp.com/attachments/380439326950948874/423482330162790400/unknown.png)
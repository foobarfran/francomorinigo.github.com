---
layout: post
title: "Dockerizando apps java para múltiples ambientes"
tags: [java, docker, maven, devops]
---
{% include JB/setup %}

**Separando configuración del código**

Una buena práctica es separar la configuración de una app que puede cambiar bastante seguido, del cóidigo (jar). Esto nos va a permitir poder cambiar la configuración de nuestra app sin tener que releasear una nueva versión o también cambiar la configuración en caliente con mucha facilidad.

{% highlight java %}
.
+-- pom.xml
+-- src
|   +-- ...
+-- conf
|   +-- common
|   |   +-- fooCommon.properties
|   +-- development
|   |   +-- logback.xml
|   |   +-- deploy-propeties.json
|   +-- staging
|   |   +-- logback.xml
|   |   +-- deploy-propeties.json
|   +-- production
|   |   +-- logback.xml
|   |   +-- deploy-propeties.json
|   +-- newrelic
|   |   +-- newrelic.yml
{% endhighlight %}


**Configurando cómo se va a levantar la JVM**

Una forma es dejarlo en un archivo, también se podrían usar variables de entorno. En este ejemplo uso el deploy-properties.json

{% highlight json %}
{
    "jvmArguments": "-Xms2048m -Xmx2048m",
    "mainClass": "com.foo.MainClass"
}
{% endhighlight %}


**NewRelic**

En el caso de usar newrelic se puede crear la carpeta newrelic para dejar la configuración. El newrelic.yml quedaría parecido a:

{% highlight yaml %}
common: &default_settings
  app_name: fooApp
staging:
  <<: *default_settings
  license_key: 'fooLicenseStaging'
production:
  <<: *default_settings
  license_key: 'fooLicenseProduction'
{% endhighlight %}

y en el pom agregamos como dependencia el agente:

{% highlight xml %}
<dependency>
    <groupId>com.newrelic.agent.java</groupId>
    <artifactId>newrelic-agent</artifactId>
    <version>${newrelic.version}</version>
</dependency>
{% endhighlight %}


**Creando una imagen base de docker a partir de la cual se corra la app con el ambiente indicado**

Simplemente creamos una imagen custom de java que tenga un script para levantar nuestra app incluyendo en el classpath en el environment que le indiquemos.

Dockerfile

{% highlight docker %}
FROM java:8

# Install jq
RUN apt-get update && \
    apt-get install -y wget && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /bin
RUN wget "http://stedolan.github.io/jq/download/linux64/jq" && chmod 755 jq

WORKDIR /

# Copy java startup script
COPY startup.sh startup.sh
RUN chmod 777 startup.sh

ENTRYPOINT ["./startup.sh"]
CMD ["development"]
EXPOSE 8080
{% endhighlight %}


Script de startup
{% highlight bash %}

#!/bin/bash
ENVIROMENT=$1
APP_PATH="application"
JAR=$(echo ${APP_PATH}/*jar)
MAINCLASS=$(jq --raw-output '.mainClass' $APP_PATH/${ENVIROMENT}/deploy-properties.json)
JVMARGS=$(jq --raw-output '.jvmArguments' $APP_PATH/${ENVIROMENT}/deploy-properties.json)

java $JVMARGS -cp "${JAR}:${APP_PATH}/common:${APP_PATH}/${ENVIROMENT}" -Dserver.port=8080 -javaagent:${APP_PATH}/newrelic/newrelic.jar -Dnewrelic.environment=${ENVIROMENT} ${MAINCLASS}

{% endhighlight %}



**Una vez que tenemos todo esto, lo único que falta es que cuando se haga un "mvn deploy" se pushee la imagen docker de nuestra app a un registry que definamos**

Agregamos las credenciales de nuestro registry a la configuración de maven:
{% highlight xml %}
<servers>
  ...
  <server>
    <id>foo-docker</id>
    <username>fooUser</username>
    <password>fooPassword</password>
    <configuration>
      <email>fooUser@email.com</email>
    </configuration>
  </server>
</servers>

{% endhighlight %}

y por último la configuración a nuestro proyecto

{% highlight xml %}
<properties>
  <docker.image.name>fooOrganization/fooApp</docker.image.name>
  <docker.resgistry.url>docker.foo.com:5000</docker.resgistry.url>
</properties>
{% endhighlight %}

{% highlight xml %}
<plugin>
  <!-- downloading newrelic agent -->
  <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>2.6</version>
    <executions>
      <execution>
        <id>copy</id>
        <phase>package</phase>
        <goals>
          <goal>copy</goal>
        </goals>
        <configuration>
          <artifactItems>
            <artifactItem>
              <groupId>com.newrelic.agent.java</groupId>
              <artifactId>newrelic-agent</artifactId>
              <version>${newrelic.version}</version>
              <type>jar</type>
              <outputDirectory>${project.build.directory}/newrelic</outputDirectory>
              <destFileName>newrelic.jar</destFileName>
            </artifactItem>
          </artifactItems>
          <outputDirectory>/newrelic</outputDirectory>
        </configuration>
      </execution>
    </executions>
  </plugin>

  <!-- building docker image -->
  <plugin>
    <groupId>com.spotify</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.4.5</version>
    <configuration>
      <serverId>docker-foo</serverId>
      <registryUrl>${docker.resgistry.url}</registryUrl>
      <baseImage>${docker.resgistry.url}/fooOrganization/java:8</baseImage>
      <imageName>${docker.image.name}</imageName>
      <imageTags>
        <imageTag>${project.version}</imageTag>
      </imageTags>
      <resources>
        <!-- adding app jar -->
        <resource>
          <targetPath>application/</targetPath>
          <directory>${project.build.directory}</directory>
          <include>${project.build.finalName}.jar</include>
        </resource>
        <!-- adding newrelic conf -->
        <resource>
          <targetPath>application/</targetPath>
          <directory>${basedir}/conf</directory>
          <include>newrelic/*.*</include>
        </resource>
        <!-- adding newrelic agent jar -->
        <resource>
          <targetPath>application/</targetPath>
          <directory>${project.build.directory}</directory>
          <include>newrelic/*.*</include>
        </resource>
        <!-- adding development conf -->
        <resource>
          <targetPath>application/</targetPath>
          <directory>${basedir}/conf</directory>
          <include>development/*.*</include>
        </resource>
        <!-- adding staging conf -->
        <resource>
          <targetPath>application/</targetPath>
          <directory>${basedir}/conf</directory>
          <include>staging/*.*</include>
        </resource>
        <!-- adding production conf -->
        <resource>
          <targetPath>application/</targetPath>
          <directory>${basedir}/conf</directory>
          <include>production/*.*</include>
        </resource>
        <!-- adding common conf -->
        <resource>
          <targetPath>application/</targetPath>
          <directory>${basedir}/conf</directory>
          <include>common/*.*</include>
        </resource>
      </resources>
    </configuration>
    <executions>
      <execution>
        <id>build-image</id>
        <phase>package</phase>
        <goals>
          <goal>build</goal>
        </goals>
      </execution>
      <execution>
        <id>tag-image</id>
        <phase>package</phase>
        <goals>
          <goal>tag</goal>
        </goals>
        <configuration>
          <image>${docker.image.name}:${project.version}</image>
          <newName>${docker.resgistry.url}/${docker.image.name}:${project.version}</newName>
        </configuration>
      </execution>
      <execution>
        <id>push-image</id>
        <phase>deploy</phase>
        <goals>
          <goal>push</goal>
        </goals>
        <configuration>
          <imageName>${docker.resgistry.url}/${docker.image.name}:${project.version}</imageName>
        </configuration>
      </execution>
    </executions>
  </plugin>

</plugins>
{% endhighlight %}

**Listo, ahora cualquiera puede correr la app dentro de un container de docker**

docker run -p 8080:8080 docker.foo.com:5000/fooOrganization/fooApp:1.0.0 [development \| staging \| production]

{% include twitter_plug.html %}

[JeKa](https://jeka.dev) is a modern Java build tool focused on simplicity.
This post demonstrates how to publish to Maven Central with minimal configuration.

**Prerequisite:** You need an [OSSRH account](https://central.sonatype.org/register/legacy/#create-an-account) to publish on Maven Central.

With JeKa, you can fully configure the build by editing the *jeka.properties* file as follows:

```properties
jeka.version=0.11.11
jeka.java.version=17

jeka.inject.classpath=dev.jeka:nexus-plugin
@nexus=

@project.moduleId=com.github.djeang:vincer-dom
@project.gitVersioning.enable=true

# Configuration for deploying to Maven central
@maven.publication.predefinedRepo=OSSRH
@maven.publication.metadata.projectName=Vincer-Dom
@maven.publication.metadata.projectDescription=Modern Dom manipulation library for Java
@maven.publication.metadata.projectUrl=https://github.com/djeang/vincer-dom
@maven.publication.metadata.projectScmUrl=https://github.com/djeang/vincer-dom.git
@maven.publication.metadata.licenses=Apache License V2.0:https://www.apache.org/licenses/LICENSE-2.0.html
@maven.publication.metadata.developers=djeang:djeangdev@yahoo.fr
```
Note that dependencies are listed in a dedicated *dependencies.txt* file to maintain clear separation of concerns.

To publish to Maven Central, execute: `jeka project:pack maven:publish`.

See a concrete example [here](https://github.com/djeang/vincer-dom).

### Explanations

Now that you know how to do it, let's explain how it works.

#### Jeka and Java Versioning

For better portability and reproducibility, we can declare both the Jeka and Java versions required for building. Both versions will be automatically downloaded if not already present on the host machine.

#### Specify Module ID and Versioning

The published `moduleId` is specified using the `@project.moduleId` property.  
The version can be explicitly specified using the `@project.version` property. Note that properties can be set in the *jeka.properties* file or passed as a command-line argument: `-D@project.version=1.0.1`.  
Instead, we choose to rely on Git to infer the version using: `@project.gitVersioning.enable=true`. If there is no tag on the current commit, the version will be set to *[branch]-SNAPSHOT*; otherwise, it will be the *tag-name*.

#### Configure Publication Repository

`@maven.publication.predefinedRepo=OSSRH` instructs Jeka to publish to the pre-defined *OSSRH* repository. This repository is configured to publish to the [OSSRH snapshot repository](https://s01.oss.sonatype.org/content/repositories/snapshots/) when the version ends with `-SNAPSHOT`, and to the [release repository](https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/) otherwise.  
The repository uses the following environment variables to pass secrets:

- `jeka.repos.publish.username`: OSSRH account username
- `jeka.repos.publish.password`: OSSRH account password
- `jeka.gpg.secret-key`: Armored GPG secret key as a string
- `jeka.gpg.passphrase`: The passphrase protecting the secret key

The content of `jeka.gpg.secret-key` can be obtained by executing: `gpg --export-secret-key --armor my-key-name`.

#### Provide Publication Metadata

The mandatory metadata are set using `@maven.publication.metadata.xxx` properties.  
Note that the `@maven.publication.metadata.licenses` property expects a format like: *[license1 name]:[license1 url],[license2 name]:[license2 url],...*

#### Automate Release Publication

For convenience, we use the [*Nexus* plugin](https://github.com/jeka-dev/jeka/tree/master/plugins/dev.jeka.plugins.nexus),  
which automatically closes the staging repository without requiring manual intervention.  
`jeka.inject.classpath=dev.jeka:nexus-plugin` instructs Jeka to fetch the plugin from Maven Central, while `@nexus=` activates it.

### Execute Build

To publish, simply execute: `jeka project:pack maven:publish`.

This will:
- Create the JAR to publish (`project: pack`).
- Create source and Javadoc JARs.
- Generate the published POM file.
- Compute all checksums.
- Sign all published files.
- Push everything to the OSSRH repository.
 
To see what will be published, execute: `jeka maven: info`.

### Fine-Tuning

Fine-tuning in Jeka is generally achieved programmatically, complementing the declarative configuration from the *jeka.properties* file. This approach allows for highly flexible and powerful configurations with the benefits of static typing.

#### Customize Transitive Dependencies

We can customize the dependencies mentioned in the published POM.  
In the following example, we remove the *com.google.guava:guava* dependency and force the *jfiglet* dependency to have the `RUNTIME` scope.

```java
class Build extends KBean {

    @Override
    void init() {
        var publication = load(MavenKBean.class).getMavenPublication();
        publication.customizeDependencies(deps -> deps
                .minus("com.google.guava:guava")
                .withTransitivity("com.github.lalyos:jfiglet", JkTransitivity.RUNTIME)
        );
    }

}
```
#### Add Extra Artifacts

The API enables defining additional artifacts to publish.  
In the following example, two artifacts are generated at publication time:

1. A ZIP file containing documentation.
2. A shaded uber JAR (a JAR that includes all classes from dependencies with renamed packages to avoid conflicts).


```java
class Build extends KBean {

    @Override
    void init() {
        var publication = load(MavenKBean.class).getMavenPublication();
        publication.putArtifact(JkArtifactId.of("doc", "zip"), this::genDoc);
        publication.putArtifact(JkArtifactId.of("shade", "jar"), project.packaging::createShadeJar);
    }

    private void genDoc(Path targetZipFile) {
        // generate documentation and zip it in targetZipFile
    }
    
}
```

#### Publish on Other Repositories

To publish to a repository other than Maven Central, you can set the following properties:

```properties
jeka.repos.publish=https://my.company/myrepo

# Optional properties
jeka.repos.publish.username=myUsername
jeka.repos.publish.password=myPassword
jeka.repos.publish.headers.Authorization=Bearer:: XHrU8hHKJHJ454==67g
```

Place these properties in *[USER HOME]/.jeka/global.properties* (instead of *jeka.properties* file) to keep configurations consistent across projects and avoid redundancy.  
For more details, refer to the [documentation](https://jeka-dev.github.io/jeka/reference/properties/#repositories).  

### Comparison with Maven

The following is the Maven POM configuration equivalent for deploying a project to Maven Central:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <developers>
        <developer>
            <name>djeang</name>
            <email>djeangdev@yahoo.fr</email>
        </developer>
    </developers>

    <name>Vincer-DOM</name>
    <description>Modern Dom manipulation library for Java.</description>
    <url>https://github.com/djeang/vincer-dom</url>

    <groupId>com.github.djeang</groupId>
    <artifactId>vincer-dom</artifactId>
    <version>0.1.0-SNAPSHOT</version>

    <licenses>
        <license>
            <name>MIT License</name>
            <url>http://www.opensource.org/licenses/mit-license.php</url>
        </license>
    </licenses>

    <properties>
        <java.version>17</java.version>
        <maven-compiler-plugin.version>3.11.0</maven-compiler-plugin.version>
    </properties>

    <scm>
        <connection>scm:git:git://github.com/djeang/vincer-dom.git</connection>
        <developerConnection>scm:git:https://github.com/djeang/vincer-dom.git</developerConnection>
        <url>https://github.com/djeang/vincer-domr</url>
    </scm>

    <distributionManagement>
        <snapshotRepository>
            <id>ossrh</id>
            <url>https://s01.oss.sonatype.org/content/repositories/snapshots</url>
        </snapshotRepository>
        <repository>
            <id>ossrh</id>
            <url>https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/</url>
        </repository>
    </distributionManagement>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-source-plugin</artifactId>
                    <version>2.2.1</version>
                    <executions>
                        <execution>
                            <id>attach-sources</id>
                            <goals>
                                <goal>jar-no-fork</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-javadoc-plugin</artifactId>
                    <version>2.9.1</version>
                    <executions>
                        <execution>
                            <id>attach-javadocs</id>
                            <goals>
                                <goal>jar</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-gpg-plugin</artifactId>
                    <version>1.5</version>
                    <executions>
                        <execution>
                          <id>sign-artifacts</id>
                          <phase>verify</phase>
                          <goals>
                              <goal>sign</goal>
                          </goals>
                        </execution>
                    </executions>
                </plugin>
                <plugin>
                    <groupId>org.sonatype.plugins</groupId>
                    <artifactId>nexus-staging-maven-plugin</artifactId>
                    <version>1.6.7</version>
                    <extensions>true</extensions>
                    <configuration>
                        <serverId>ossrh</serverId>
                        <nexusUrl>https://s01.oss.sonatype.org/</nexusUrl>
                        <autoReleaseAfterClose>true</autoReleaseAfterClose>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>me.ccampo</groupId>
                <artifactId>git-version-maven-plugin</artifactId>
                <version>0.1.0</version> <!-- Use the latest stable version if possible -->
                <extensions>true</extensions>
                <configuration>
                    <strategy hint="git">
                        <!-- Strategy specific configuration goes here -->
                    </strategy>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

### Conclusion

Jeka provides a simpler, yet powerful way to build Java software and publish artifacts to Maven Central or other repositories, with much less configuration and effort than traditional tools.

Visit the [website](https://jeka.dev), [videos](https://jeka.dev/#seeinaction), and [examples](https://jeka-dev.github.io/jeka/examples/) to get an idea of everything Jeka can do better.

*Disclaimer:* I am the author of Jeka.

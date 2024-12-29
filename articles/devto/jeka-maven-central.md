[JeKa](https://jeka.dev) is a modern Java build tool focusing on simplicity.
This post showcases how to publish on Maven Central with very few configuration.

**Prerequisite:** You must be have an [OSSRH account](https://central.sonatype.org/register/legacy/#create-an-account).

Using Jeka, the build can be entirely configured by editing the *jeka.properties* file as follow:

```properties
jeka.version=0.11.11
jeka.java.version=8

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
### Explanations

#### Specify Module Id and Versioning

The published moduleId is specified using the `@project.muduleId` property.
The version can be specified using the `@project.version` property. Note that properties can be set in the *jeka.properties* file or passed in command line argument as: `-D@project.version=1.0.1`.
Here we decide to rely on Git to infer version: `@project.gitVersioning.enable=true`. If there is no tag on current commit, version will value *[branch]-SNAPSHOT* and *tag-name* otherwise.

#### Configure Publication Repo

`@maven.publication.predefinedRepo=OSSRH` instucts Jeka to publish on the pre-defined `OSSRH` repository which is configured to publish on the [OSSRH-snapshot repository](https://s01.oss.sonatype.org/content/repositories/snapshots/) when version ends with `-SNAPSHOT` and the [release one](https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/) in other cases.
The repo use the following environment variables to pass secrets:

- `jeka.repos.publish.username`: OSSRH account username
- `jeka.repos.publish.password`: OSSRH account password
- `jeka.gpg.secret-key`: Armored GPG secret key as string
- `jeka.gpg.passphrase`: The passphrase protecting the secret-key.

The content of `jeka.gpg.secret-key` has been obtained by executing : `gpg --export-secret-key --armor my-key-name`.

#### Feed Publication Metadata

The mandatory metadata are set using `@maven.publication.metadata.xxx` properties.
Nota that the `@maven.publication.metadata.licenses` property expects a format as: *[licence1 name]:[license1 url],[licence2 name]:[license2 url],...*

#### Automate Release Publication

For convenience, we use the [*Nexus* plugin](https://github.com/jeka-dev/jeka/tree/master/plugins/dev.jeka.plugins.nexus), 
that closes automatically staging repository without needing manual intervention.
`jeka.inject.classpath=dev.jeka:nexus-plugin` instructs Jeka to fetch the plugin on Maven central, while `@naxus=` activate it.
@nexus=

### Execute Build

To publish, just execute `jeka project: pack maven: publish`.
This will:
  - Create jar to publish (`project: pack`).
  - Create sources and javadoc jars?
  - Create published pom.
  - Compute all check sums.
  - Sign all published files.
  - Push everything to OSSRH repository.

To see what is 

### Fine Tuning

Fine-tuning, in Jeka, is generally achieved programmatically, completing the declarative configuration from *jeka.properties* file.
This allow super-flexible and powerfull configuration with the benefit of the 
static typing.

To see details on what will be published, execute: `jeka maven: info`.

#### Customize Transitive Dependencies

We can customize dependencies mentioned in the published POM.
In the following example, we remove *com.google.guava:guava* dependency, and 
we force the the *jfiglet* dependency to have the RUNTIME scope.
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

The API allows to define extra artifact to publish. In the following example, two artifacts 
are created at publication time: a zip file containing documentation and 
a shade uber jar (a jar containing all classes from dependencies with renamed packages to avoid collisions.)


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


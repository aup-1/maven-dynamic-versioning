## Trunk based development, Dynamic versioning with Maven

For [trunk based development], every commit to master needs to be versioned, and deployed through CI/CD pipelines.
The version each commit gets needs to be unique, and we should be able to correlate which version got deployed has 
which changes. Git commit hash is unique for each commit, so we can use that to correlation.
However, setting this version in very straight forward with maven.

Following are few ideas and tools on how to do such dynamic versioning with maven.


#### git-commit-id-maven-plugin
This plugin writes the git commit information to `git.properties` file at build time, 
which you can use for versioning.All these properties are available to you as project properties. 
You can use these properties directly, to set your project version, if you are working on an independent 
project. 

#### gmavenplus-plugin

If you have a requirement where all your child projects needs to follow the same version scheme,
in your maven parent file, you can take advantage of `gmavenplus-plugin` scripting section to 
achieve this. But, caveat is that, you cannot change the project version dynamically inside maven pom.
We can set version externally using `mvn versions:set` with `newVersion` property. 
We can take advantage of `Flatten Maven Plugin` using which we can install and publish the modified 
version of our project version.

```xml
           <plugin>
                <groupId>org.codehaus.gmavenplus</groupId>
                <artifactId>gmavenplus-plugin</artifactId>
                <version>${mvn-gmavenplus.version}</version>
                <executions>
                    <execution>
                        <id>change-version</id>
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>execute</goal>
                        </goals>
                        <configuration>
                            <scripts>
                                <script>
                                    import org.apache.maven.artifact.versioning.VersionRange
                                    def git_revision = project.properties['git.commit.id.abbrev']
                                    def count = project.properties['git.total.commit.count']
                                    def dirty = Boolean.parseBoolean(project.properties['git.dirty'])
                                    def version = dirty ? 1 + "." +  count + ".dirty" :
                                    1 + "." +  count + "." + git_revision

                                    <!--   consolidate user and project properties-->
                                    def mvnProperties = project.properties + session.userProperties

                                    project.artifact.version = version
                                    project.artifact.versionRange = VersionRange.createFromVersion(version)
                                    project.version = version
                                </script>
                            </scripts>
                        </configuration>
                    </execution>
                </executions>
                <dependencies>
                    <dependency>
                        <groupId>org.codehaus.groovy</groupId>
                        <artifactId>groovy-all</artifactId>
                        <version>${groovy-all.version}</version>
                        <scope>runtime</scope>
                        <type>pom</type>
                    </dependency>
                </dependencies>
            </plugin>

```
 Flatten configuration
 
 ```xml
        <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>flatten-maven-plugin</artifactId>
                <version>${maven-flatten.version}</version>
                <configuration>
                    <flattenedPomFilename>.flattened-pom.xml</flattenedPomFilename>
                    <outputDirectory>${project.build.directory}</outputDirectory>
                    <updatePomFile>true</updatePomFile>
                    <flattenMode>resolveCiFriendliesOnly</flattenMode>
                    <pomElements>
                        <profiles/>
                    </pomElements>
                </configuration>
                <executions>
                    <execution>
                        <id>flatten</id>
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>flatten</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>flatten.clean</id>
                        <phase>clean</phase>
                        <goals>
                            <goal>clean</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

```
  
### 
If you have the flexibility to derive this version externally, setting the new version is much easier.
If you are using gitlab, git commit information is made available  
https://docs.gitlab.com/ee/ci/variables/predefined_variables.html

One can take advantage of maven CI friendly variables for setting the dynamic version. 


```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.apache</groupId>
    <artifactId>apache</artifactId>
    <version>18</version>
  </parent>
  <groupId>org.apache.maven.ci</groupId>
  <artifactId>ci-parent</artifactId>
  <name>First CI Friendly</name>
  <version>${revision}</version>
  ...
</project>

```
this in combination with maven `flatten plugin` as show above, you should be able to install and publish the new version.

with 
```
mvn -Drevision=${newversion} clean package
```

#### Links:

- [git-commit-id-maven-plugin](https://github.com/git-commit-id/git-commit-id-maven-plugin)
- [trunk based development](https://trunkbaseddevelopment.com/)
- [maven flatten plugin](https://www.mojohaus.org/flatten-maven-plugin/)
- [maven ci friendly](https://maven.apache.org/maven-ci-friendly.html)


[trunk based development]: (https://trunkbaseddevelopment.com/)
[maven ci friendly]: (https://maven.apache.org/maven-ci-friendly.html)
[git-commit-id-maven-plugin]: (https://github.com/git-commit-id/git-commit-id-maven-plugin)
[maven flatten plugin]:(https://www.mojohaus.org/flatten-maven-plugin/)


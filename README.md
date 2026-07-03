# Spring Boot Build Image Bind Cache Reproducer

This reproducer shows that Spring Boot Maven Plugin `build-image` uses bind-mounted cache directories, but a second build still re-downloads the BellSoft Liberica JRE layer instead of reusing it.

## Expected behavior

On the second build, after:

- bind cache directories are populated by the first build
- the previously built app image is removed

The BellSoft Liberica JRE layer should be reused from the bind cache.

Expected log shape:

```text
BellSoft Liberica JRE ...: Reusing cached layer
```

## Actual behavior

On the second build:

- bind cache is definitely used
- some cache entries are restored (`syft`, `spring-cloud-bindings`, `SBOM`)
- but BellSoft Liberica JRE is downloaded again

Actual log shape:

```text
Using build cache bind mount '/tmp/spring-boot-build-image-cache-repro-cache/build'
...
Image with name "docker.io/library/spring-boot-build-image-cache-repro:0.0.1-SNAPSHOT" not found
...
BellSoft Liberica JRE 25.0.3: Contributing to layer
Downloading from https://github.com/bell-sw/Liberica/releases/download/25.0.3+11/bellsoft-jre25.0.3+11-linux-aarch64.tar.gz
```

## Project layout

```text
spring-boot-build-image-cache-repro/
├── pom.xml
└── src/main/java/com/example/DemoApplication.java
```

## Step By Step

1. Remove the previous app image:

```bash
docker image rm docker.io/library/spring-boot-build-image-cache-repro:0.0.1-SNAPSHOT 2>/dev/null || true
```

2. Remove previous Paketo Docker volumes:

```bash
docker volume ls --format '{{.Name}}' | grep '^pack-cache' | xargs docker volume rm 2>/dev/null || true
```

3. Remove previous bind cache directories and recreate them:

```bash
rm -rf /tmp/spring-boot-build-image-cache-repro-cache
mkdir -p /tmp/spring-boot-build-image-cache-repro-cache/work
mkdir -p /tmp/spring-boot-build-image-cache-repro-cache/build
mkdir -p /tmp/spring-boot-build-image-cache-repro-cache/launch
```

4. Run the first build:

```bash
./mvnw -DskipTests clean package spring-boot:build-image
```

5. Confirm the cache directories were populated:

```bash
ls -la /tmp/spring-boot-build-image-cache-repro-cache/build
ls -la /tmp/spring-boot-build-image-cache-repro-cache/launch
```

Expected shape:

```text
... committed/
```

6. Remove the app image again so the second build cannot reuse from the app image:

```bash
docker image rm docker.io/library/spring-boot-build-image-cache-repro:0.0.1-SNAPSHOT
```

7. Run the second build:

```bash
./mvnw -DskipTests clean package spring-boot:build-image
```

8. Inspect the second build output.

The important evidence is:

- bind cache is used:

```text
Using build cache bind mount '/tmp/spring-boot-build-image-cache-repro-cache/build'
```

- app image is absent:

```text
Image with name "docker.io/library/spring-boot-build-image-cache-repro:0.0.1-SNAPSHOT" not found
```

- some cache entries are restored:

```text
Restoring metadata for "paketo-buildpacks/syft:syft" from cache
Restoring metadata for "paketo-buildpacks/spring-boot:spring-cloud-bindings" from cache
Restoring data for "paketo-buildpacks/syft:syft" from cache
Restoring data for "paketo-buildpacks/spring-boot:spring-cloud-bindings" from cache
Restoring data for SBOM from cache
```

- but Liberica is still downloaded:

```text
BellSoft Liberica JRE 25.0.3: Contributing to layer
Downloading from https://github.com/bell-sw/Liberica/releases/download/25.0.3+11/bellsoft-jre25.0.3+11-linux-aarch64.tar.gz
```

## pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.15</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>spring-boot-build-image-cache-repro</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <properties>
        <java.version>17</java.version>
        <pack.cache.root>/tmp/spring-boot-build-image-cache-repro-cache</pack.cache.root>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <image>
                        <name>docker.io/library/spring-boot-build-image-cache-repro:0.0.1-SNAPSHOT</name>
                        <buildWorkspace>
                            <bind>
                                <source>${pack.cache.root}/work</source>
                            </bind>
                        </buildWorkspace>
                        <buildCache>
                            <bind>
                                <source>${pack.cache.root}/build</source>
                            </bind>
                        </buildCache>
                        <launchCache>
                            <bind>
                                <source>${pack.cache.root}/launch</source>
                            </bind>
                        </launchCache>
                    </image>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

## DemoApplication.java

```java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

## Conclusion

Bind-mounted `buildCache` / `launchCache` directories are clearly used and do restore some buildpack cache entries, but they do not appear to restore the BellSoft Liberica JRE layer for reuse on the second build.

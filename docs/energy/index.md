# Galacticraft Energy
An energy API that utilizes LibBlockAttributes.

## Installing
![Maven metadata URL](https://img.shields.io/maven-metadata/v?logo=Apache%20Maven&metadataUrl=https%3A%2F%2Fmaven.galacticraft.dev%2Fdev%2Fgalacticraft%2FGalacticraftEnergy%2Fmaven-metadata.xml&style=flat-square&logoColor=white)

=== "Groovy (`build.gradle`)"
    ```groovy
    repositories {
        maven {
            url "https://maven.galacticraft.dev"
            content {
                includeGroup("dev.galacticraft")
            }
        }
    }

    dependencies {
        include modApi "dev.galacticraft:GalacticraftEnergy:$version"
    }
    ```

=== "Kotlin DSL (`build.gradle.kts`)"
    ```kotlin
    repositories {
        maven("https://maven.galacticraft.dev") {
            content {
                includeGroup("dev.galacticraft")
            }
        }
    }

    dependencies {
        include(modApi("dev.galacticraft:GalacticraftEnergy:$version") {})
    }
    ```

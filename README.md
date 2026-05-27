# CI Workflows

Reusable GitHub Actions workflows para los proyectos de **icgdesarrollo**, con análisis de [SonarCloud](https://sonarcloud.io/organizations/icgdesarrollo) y reporte de cobertura.

## ¿Qué resuelve este repo?

Configurar SonarCloud y CI en un proyecto nuevo solía requerir copiar y pegar 30+ líneas de YAML, configurar plugins de cobertura y mantener todo sincronizado entre repos. Este repo centraliza esa lógica:

- **Una sola fuente de verdad** para el flujo de análisis
- **10 líneas de YAML** en cada repo nuevo, en lugar de 30
- **Mantenimiento centralizado**: actualizar la versión del scanner aquí actualiza todos los repos

## Workflows disponibles

| Workflow | Stack |
|---|---|
| `sonar-java-maven.yml` | Java + Maven |
| `sonar-node.yml` | Node.js (JavaScript / TypeScript) |
| `sonar-python.yml` | Python |

## ¿Dónde se configura Sonar en cada stack?

La configuración de Sonar vive en un archivo distinto según el stack. Esto es importante porque el scanner solo lee el archivo correspondiente a tu stack:

| Stack | Archivo de configuración |
|---|---|
| Java + Maven | Propiedades dentro de `pom.xml` |
| Node.js | `sonar-project.properties` en la raíz |
| Python | `sonar-project.properties` en la raíz |

> **Nota para Java**: técnicamente el scanner puede leer `sonar-project.properties`, pero los plugins de Maven inyectan las propiedades del `pom.xml` como argumentos CLI, que tienen **mayor prioridad**. Si tienes ambos, el `.properties` queda como código muerto y se desincroniza fácilmente. La convención en Java es configurar Sonar dentro del build tool, así que evita crear `sonar-project.properties` en estos proyectos.

## Pre-requisitos comunes (todos los stacks)

Antes de usar cualquier workflow:

1. **El proyecto debe existir en SonarCloud** bajo la organización `icgdesarrollo`
   - Ve a https://sonarcloud.io/projects/create
   - Conecta el repo de GitHub
   - Anota el **Project Key** generado (ej: `icgdesarrollo_mi-repo`)

2. **Desactivar Automatic Analysis en SonarCloud**
   - Ve a tu proyecto → **Administration → Analysis Method**
   - Apaga el toggle de **Automatic Analysis**
   - Si no se desactiva, chocará con el análisis del CI y verás errores

3. **El secret `SONAR_TOKEN`** está disponible a nivel de organización
   - No es necesario crearlo en cada repo
   - Se hereda automáticamente

---

## Java + Maven

### 1. Configura tu `pom.xml`

Agrega las propiedades de Sonar dentro de `<properties>`:

```xml
<properties>
    <java.version>21</java.version>
    <sonar.projectKey>icgdesarrollo_NOMBRE_DEL_REPO</sonar.projectKey>
    <sonar.organization>icgdesarrollo</sonar.organization>
    <sonar.host.url>https://sonarcloud.io</sonar.host.url>
    <sonar.coverage.jacoco.xmlReportPaths>target/site/jacoco/jacoco.xml</sonar.coverage.jacoco.xmlReportPaths>
</properties>
```

Agrega el plugin de JaCoCo dentro de `<build><plugins>`:

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution>
            <id>prepare-agent</id>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### 2. Crea `.github/workflows/build.yml`

```yaml
name: Build & SonarCloud

on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  sonar:
    uses: icgdesarrollo/ci-workflows/.github/workflows/sonar-java-maven.yml@main
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### 3. Si necesitas otra versión de Java

```yaml
jobs:
  sonar:
    uses: icgdesarrollo/ci-workflows/.github/workflows/sonar-java-maven.yml@main
    with:
      java-version: '17'
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### 4. Verifica el reporte de cobertura

Ejecuta localmente antes del primer push:

```bash
mvn clean verify
```

Confirma que existe el archivo `target/site/jacoco/jacoco.xml`. Si no existe, JaCoCo no se está ejecutando.

---

## Node.js (JavaScript / TypeScript)

### 1. Configura tu test runner para generar cobertura `lcov`

#### Si usas **Jest**, en tu `package.json`:

```json
{
  "scripts": {
    "test": "jest --coverage"
  },
  "jest": {
    "coverageReporters": ["lcov", "text"],
    "collectCoverageFrom": [
      "src/**/*.{js,jsx,ts,tsx}",
      "!src/**/*.d.ts"
    ]
  }
}
```

#### Si usas **Vitest**, en `vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      include: ['src/**/*.{js,jsx,ts,tsx}'],
    },
  },
})
```

Y en tu `package.json`:

```json
{
  "scripts": {
    "test": "vitest run --coverage"
  }
}
```

### 2. Crea `sonar-project.properties` en la raíz del repo

```properties
sonar.projectKey=icgdesarrollo_NOMBRE_DEL_REPO
sonar.organization=icgdesarrollo

sonar.sources=src
sonar.tests=src
sonar.test.inclusions=**/*.test.ts,**/*.test.tsx,**/*.test.js,**/*.test.jsx,**/*.spec.ts,**/*.spec.tsx,**/*.spec.js,**/*.spec.jsx
sonar.exclusions=**/node_modules/**,**/dist/**,**/build/**,**/coverage/**

sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.typescript.lcov.reportPaths=coverage/lcov.info
sonar.sourceEncoding=UTF-8
```

A diferencia de Java, Node sí lee este archivo automáticamente.

### 3. Crea `.github/workflows/build.yml`

#### Con **npm** (default):

```yaml
name: Build & SonarCloud

on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  sonar:
    uses: icgdesarrollo/ci-workflows/.github/workflows/sonar-node.yml@main
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

#### Con **yarn**:

```yaml
jobs:
  sonar:
    uses: icgdesarrollo/ci-workflows/.github/workflows/sonar-node.yml@main
    with:
      package-manager: yarn
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

#### Con **pnpm**:

```yaml
jobs:
  sonar:
    uses: icgdesarrollo/ci-workflows/.github/workflows/sonar-node.yml@main
    with:
      package-manager: pnpm
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### 4. Si necesitas otra versión de Node o un comando de test diferente

```yaml
jobs:
  sonar:
    uses: icgdesarrollo/ci-workflows/.github/workflows/sonar-node.yml@main
    with:
      node-version: '18'
      test-command: 'test:ci'  # se ejecutará como `npm run test:ci`
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### 5. Verifica el reporte de cobertura

Localmente:

```bash
npm test
```

Confirma que existe `coverage/lcov.info`.

---

## Python

### 1. Configura las dependencias de testing

Agrega a tu `requirements-dev.txt` (o donde tengas las dependencias de desarrollo):

```
pytest
pytest-cov
```

### 2. Configura `pytest` para generar cobertura

Crea o actualiza `pytest.ini` (o `pyproject.toml`):

#### Opción A: `pytest.ini`

```ini
[pytest]
addopts = --cov=. --cov-report=xml --cov-report=term
testpaths = tests
```

#### Opción B: `pyproject.toml`

```toml
[tool.pytest.ini_options]
addopts = "--cov=. --cov-report=xml --cov-report=term"
testpaths = ["tests"]

[tool.coverage.run]
omit = [
    "tests/*",
    "*/migrations/*",
    "venv/*",
]
```

### 3. Crea `sonar-project.properties` en la raíz

```properties
sonar.projectKey=icgdesarrollo_NOMBRE_DEL_REPO
sonar.organization=icgdesarrollo

sonar.sources=.
sonar.tests=tests
sonar.exclusions=**/venv/**,**/__pycache__/**,**/migrations/**,**/tests/**

sonar.python.coverage.reportPaths=coverage.xml
sonar.python.version=3.12
sonar.sourceEncoding=UTF-8
```

> **Nota sobre `tests/`**: aunque `sonar.tests=tests` indica a Sonar que esa carpeta contiene tests, también la incluimos en `sonar.exclusions` para que no se analice como código productivo. Sonar la seguirá reconociendo como carpeta de tests gracias a `sonar.tests`.

> **Tip**: si tu código no está en la raíz sino en una carpeta como `app/` o `src/`, ajusta `sonar.sources` y el `--cov=.` del pytest a esa ruta.

### 4. Crea `.github/workflows/build.yml`

```yaml
name: Build & SonarCloud

on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  sonar:
    uses: icgdesarrollo/ci-workflows/.github/workflows/sonar-python.yml@main
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### 5. Si usas otra versión de Python o comando de test

```yaml
jobs:
  sonar:
    uses: icgdesarrollo/ci-workflows/.github/workflows/sonar-python.yml@main
    with:
      python-version: '3.11'
      test-command: 'coverage run -m unittest discover && coverage xml'
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### 6. Verifica el reporte de cobertura

Localmente:

```bash
pip install -r requirements-dev.txt
pytest
```

Confirma que existe `coverage.xml`.

---

## Monorepos y proyectos multi-módulo

Si tu repo contiene varios módulos o sub-proyectos, hay dos enfoques posibles:

### Opción A: un solo proyecto en SonarCloud (recomendado para la mayoría de casos)

Trata todo el monorepo como un único proyecto de Sonar. Solo necesitas ajustar las rutas de fuentes, tests y reportes para que apunten a todos los módulos.

**Ejemplo Node.js** (monorepo con `packages/api` y `packages/web`):

```properties
sonar.projectKey=icgdesarrollo_mi-monorepo
sonar.organization=icgdesarrollo

sonar.sources=packages/api/src,packages/web/src
sonar.tests=packages/api/src,packages/web/src
sonar.exclusions=**/node_modules/**,**/dist/**,**/coverage/**

sonar.javascript.lcov.reportPaths=packages/api/coverage/lcov.info,packages/web/coverage/lcov.info
```

**Ejemplo Maven multi-módulo**: SonarCloud lo detecta automáticamente con `mvn verify sonar:sonar` desde el `pom.xml` padre. Asegúrate de que el plugin de JaCoCo esté en el `pom.xml` padre (en `<pluginManagement>`) para que se herede a los módulos hijos.

### Opción B: un proyecto por módulo en SonarCloud

Útil si cada módulo tiene equipos o quality gates distintos. En este caso necesitas un workflow por módulo y cada uno apunta a un `projectKey` diferente. Habla con el equipo de plataforma antes de elegir esta opción para evitar fragmentar dashboards.

---

## Solución de problemas comunes

### "No data available to display" en la tarjeta de Coverage

El reporte de cobertura no llegó a SonarCloud. Verifica:

1. Los tests se ejecutaron localmente y generaron el archivo de reporte (`jacoco.xml`, `lcov.info`, `coverage.xml`)
2. La ruta en `sonar.coverage.*.reportPaths` (o `sonar.javascript.lcov.reportPaths`, etc.) es correcta
3. El log del workflow en GitHub Actions muestra que el archivo se subió: busca líneas tipo `Sensor JaCoCo XML Report Importer`

### "You are running CI analysis while Automatic Analysis is enabled"

Desactiva Automatic Analysis en SonarCloud → Administration → Analysis Method.

### "Project not found" o "Organization not found"

Revisa que `sonar.projectKey` y `sonar.organization` coincidan exactamente con los valores que ves en SonarCloud (Project Information). Son case-sensitive.

### El test `xxx` falla en CI pero pasa local

Suele ser por dependencias del filesystem (rutas con `\` en Windows) o de variables de entorno. El runner de GitHub Actions corre en Ubuntu por defecto.

### "Circular dependency detected" en Gradle

Asegúrate de no tener `dependsOn(tasks.test)` dentro de `jacocoTestReport` y `finalizedBy(tasks.jacocoTestReport)` dentro de `test` al mismo tiempo. Deja solo `finalizedBy` en la tarea `test`.

---

## Mantenimiento de este repo

Si necesitas actualizar la versión del SonarScanner, agregar steps, o cambiar la versión por defecto del runtime:

1. Edita el workflow correspondiente
2. Haz push a `main`
3. Todos los repos que apunten a `@main` heredarán el cambio en la siguiente ejecución

> **Tip**: si en algún momento necesitas estabilidad y no quieres que un cambio aquí afecte a otros repos automáticamente, puedes pinear desde el repo consumidor a un tag específico:
>
> ```yaml
> uses: icgdesarrollo/ci-workflows/.github/workflows/sonar-java-maven.yml@v1.0.0
> ```

## Recursos

- [SonarCloud — icgdesarrollo](https://sonarcloud.io/organizations/icgdesarrollo)
- [Documentación SonarCloud](https://docs.sonarsource.com/sonarqube-cloud/)
- [Reusable workflows en GitHub Actions](https://docs.github.com/en/actions/using-workflows/reusing-workflows)

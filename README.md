# Deploy to Astro
This action deploys code from your GitHub repository to an Astro Deployment. It runs whenever you commit to your repository's main branch. For more information about Astro CI/CD workflows, see [Automate code deploys with CI/CD](https://docs.astronomer.io/astro/ci-cd).

This action can be used only to deploy code from `main` or an equivalent branch. To configure a CI/CD pipeline for multiple branches, see [Astronomer documentation](https://docs.astronomer.io/astro/ci-cd?tab=multiple%20branch#github-actions-dag-based-deploy). 

You can use the Deploy Action with DAG-only deploys activated or deactivated. When DAG-only deploys are activated, the action does not rebuild and deploy your image when your commit only includes changes to the `/dags` folder. For more information about DAG-only deploys, see [Deploy DAGs only](https://docs.astronomer.io/astro/deploy-code#deploy-dags-only).

The action completes the following steps whenever you commit to your main branch:

- Checks out your repository.
- Checks whether your commit only changed DAG code.
- Optional. Tests DAG code with pytest.
- If the commit included only changes to DAG code, pushes the change to Astro without building a new project image.
- If the change included changes to project configurations, rebuilds your project image and deploys it to Astro.

## Usage

To use the action, you must set the `ASTRONOMER_KEY_ID` and `ASTRONOMER_KEY_SECRET` environment variables in your GitHub Actions workflow to the Key ID and secret for an existing [Deployment API key](https://docs.astronomer.io/astro/api-keys). Astronomer recommends using GitHub Actions secrets to set these environment variables. An example workflow script is provided in **Workflow file example**. 

## Configuration options

The following table lists the optional configuration options for Deploy Actions.

| Name | Default | Description |
| ---|---|--- |
| `dag-deploy-enabled` | `false` | When set to `true`, DAG files are deployed only when the DAG files are changed. Only set this to `true` when DAG-only deploys are activated on the Deployment you are deploying to. |
| `root-folder` | `.` | Specifies the path to to Astro project folder containing the 'dags' folder | 
| `parse` | `false` | When set to `true`, DAGs are parsed for errors before deployment |
| `pytest` | `false` | When set to `true`, pytests are run before deployment |
| `pytest-file` | (all tests run) | Specifies the custom pytest files to run with the pytest command. For example, you could specify `/tests/test-tags.py`|
| `force` | `false` | When set to `true`, your code is force deployed without testing or parsing |
| `image-name` |  | Specifies a custom, locally built image to deploy |


## Workflow file examples


In the following example, DAG-only deploys are enabled and DAG files are parsed for both image and DAG deploys.

```
name: Astronomer CI - Deploy code

on:
  push:
    branches:
      - main

env:
  ## Sets Deployment API key credentials as environment variables
  ASTRONOMER_KEY_ID: ${{ secrets.ASTRONOMER_KEY_ID }}
  ASTRONOMER_KEY_SECRET: ${{ secrets.ASTRONOMER_KEY_SECRET }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to Astro
      uses: astronomer/deploy-action@v0.1
      with:
        dag-deploy-enabled: true
        parse: true
```

Use the following topics to further configure the action based on your needs.

### Change the DAG folder

In the following example, the folder `/example-dags/dags` is specified as the DAG folder.

```
steps:
- name: Deploy to Astro
  uses: astronomer/deploy-action@v0.1
  with:
    dag-folder: /example-dags/dags/
```

### Run Pytests

In the following example, the pytest located at `/tests/test-tags.py` runs before the image is deployed. 

```
steps:
- name: Deploy to Astro
  uses: astronomer/deploy-action@v0.1
  with:
    pytest: true
    pytest-file: /tests/test-tags.py
```

### Ignore parsing and testing

In the following example, the parse and pytests are skipped.

```
steps:
- name: Deploy to Astro
  uses: astronomer/deploy-action@v0.1
  with:
    force: true
```

### Use a custom image

In the following example, a custom image is built and deployed to an Astro Deployment.

```
name: Astronomer CI - Additional build-time args

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      ASTRONOMER_KEY_ID: ${{ secrets.ASTRO_ACCESS_KEY_ID_DEV }}
      ASTRONOMER_KEY_SECRET: ${{ secrets.ASTRO_SECRET_ACCESS_KEY_DEV }}
    steps:
    - name: Check out the repo
      uses: actions/checkout@v3
    - name: Create image tag
      id: image_tag
      run: echo ::set-output name=image_tag::astro-$(date +%Y%m%d%H%M%S)
    - name: Build image
      uses: docker/build-push-action@v2
      with:
        tags: ${{ steps.image_tag.outputs.image_tag }}
        load: true
        # Define your custom image's build arguments, contexts, and connections here using
        # the available GitHub Action settings:
        # https://github.com/docker/build-push-action#customizing .
        # This example uses `build-args` , but your use case might require configuring
        # different values.
        build-args: |
          <your-build-arguments>
    - name: Deploy to Astro
      uses: astronomer/deploy-action@v0.1
      with:
        image-name: ${{ steps.image_tag.outputs.image_tag }}
      
```

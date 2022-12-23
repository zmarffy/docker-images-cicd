# `docker-images-cicd`

`docker-images-cicd` is a Jenkins CI/CD pipeline for building and pushing Docker images.

## Requirements

You must have [zeke-jenkins-tools](https://github.com/zmarffy/zekes-jenkins-tools) as a shared library accessible by this job.

If you add a file called `cicd/docker-images.yaml` to your repo, this file will be analyzed to determine exactly how to build the images in the repo. This file should contain a list of maps which have keys of the folder the image definition is in, the build args for the image, and the tags for the image. See [`pybuilder`'s `cicd/docker-images.yaml`](https://github.com/zmarffy/pybuilder/blob/main/cicd/docker-images.yaml) file for a real-world example.

## Usage

### Parameters

- `GIT_LOCATION`: The full `git` URL of the project to build
- `GIT_TARGET_BRANCH`: The name of the branch to build the images from
- `GIT_CREDS_ID`: The Jenkins credentials ID used to clone the `git` repo
- `PUSH`: Whether or not to push the image to a registry
- `SAVE`: Whether or not to save the created Docker images to Jenkins artifacts
- `REGISTRY`: The URL of the registry to push to. If blank, Docker Hub will be used
- `REGISTRY_CREDS_ID`: The Jenkins credentials ID used to push project to the registry
- `CREATE_SINGLE_MANIFEST_FOR_ALL_ARCHS`: Whether or not to create a manifest that points to images of all built architectures (can only be used if `PUSH` is true)
- `CLEAN_UP_SINGLE_ARCH_REPOS`: Whether or not to delete the single-architecture repos when done and leave only the multi-arch manifest (can only be used if `PUSH` is true, `REGISTRY` is blank (Docker Hub), and `CREATE_SINGLE_MANIFEST_FOR_ALL_ARCHS` is true
- `IMAGES_FOLDER`: The location where specifically versioned Docker images are in repo
images
- `IMAGE_NAME`: The name of image when built. If blank, the `git` repo name will be used
- `ARCHS`: The platforms to build images for, space delimited. If blank, build on all possible architectures
- `TAGS`: The tags of images to build, space delimited. If blank, all possible tags will be built

## Future enhancements

- A testing mechanism
- Push updates for a specific arch image without wiping out the rest of the manifest

---
id: seccompv2
title: Seccomp V2
---

# Seccomp V2

## Description
Controls the seccomp profile used by containers. Corresponds to the `securityContext.seccompProfile` field. Security contexts from the annotation is not considered as Kubernetes no longer reads security contexts from the annotation.

## Template
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spspseccompv2
  annotations:
    metadata.gatekeeper.sh/title: "Seccomp V2"
    metadata.gatekeeper.sh/version: 1.0.0
    description: >-
      Controls the seccomp profile used by containers. Corresponds to the
      `securityContext.seccompProfile` field. Security contexts from the annotation is not considered as Kubernetes no longer reads security contexts from the annotation.
spec:
  crd:
    spec:
      names:
        kind: K8sPSPSeccompV2
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          type: object
          description: >-
            Controls the seccomp profile used by containers. Corresponds to the
            `securityContext.seccompProfile` field. Security contexts from the annotation is not considered as Kubernetes no longer reads security contexts from the annotation.
          properties:
            exemptImages:
              description: >-
                Any container that uses an image that matches an entry in this list will be excluded
                from enforcement. Prefix-matching can be signified with `*`. For example: `my-image-*`.

                It is recommended that users use the fully-qualified Docker image name (e.g. start with a domain name)
                in order to avoid unexpectedly exempting images from an untrusted repository.
              type: array
              items:
                type: string
            allowedProfiles:
              type: array
              description: >-
                An array of allowed profile values for seccomp on Pods/Containers.

                Can use the securityContext naming scheme: `RuntimeDefault`, `Unconfined`
                and/or `Localhost`. For securityContext `Localhost`, use the parameter `allowedLocalhostFiles`
                to list the allowed profile JSON files.

                The policy code will translate between the two schemes so it is not necessary to use both.

                Putting a `*` in this array allows all Profiles to be used.

                This field is required since with an empty list this policy will block all workloads.
              items:
                type: string
            allowedLocalhostFiles:
              type: array
              description: >-
                When using securityContext naming scheme for seccomp and including `Localhost` this array holds
                the allowed profile JSON files.

                Putting a `*` in this array will allows all JSON files to be used.

                This field is required to allow `Localhost` in securityContext as with an empty list it will block.
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      code: 
      - engine: K8sNativeValidation
        source:
          variables:
          - name: containers
            expression: 'has(variables.anyObject.spec.containers) ? variables.anyObject.spec.containers : []'
          - name: initContainers
            expression: 'has(variables.anyObject.spec.initContainers) ? variables.anyObject.spec.initContainers : []'
          - name: ephemeralContainers
            expression: 'has(variables.anyObject.spec.ephemeralContainers) ? variables.anyObject.spec.ephemeralContainers : []'
          - name: allowAllProfiles
            expression: |
              has(variables.params.allowedProfiles) && variables.params.allowedProfiles.exists(profile, profile == "*")
          - name: exemptImagePrefixes
            expression: |
              !has(variables.params.exemptImages) ? [] :
                variables.params.exemptImages.filter(image, image.endsWith("*")).map(image, string(image).replace("*", ""))
          - name: exemptImageExplicit
            expression: |
              !has(variables.params.exemptImages) ? [] : 
                variables.params.exemptImages.filter(image, !image.endsWith("*"))
          - name: exemptImages
            expression: |
              (variables.containers + variables.initContainers + variables.ephemeralContainers).filter(container,
                container.image in variables.exemptImageExplicit ||
                variables.exemptImagePrefixes.exists(exemption, string(container.image).startsWith(exemption))).map(container, container.image)
          - name: unverifiedContainers
            expression: |
              (variables.containers + variables.initContainers + variables.ephemeralContainers).filter(container,
                !variables.allowAllProfiles &&
                !(container.image in variables.exemptImages))
          - name: inputNonLocalHostProfiles
            expression: |
              variables.params.allowedProfiles.filter(profile, profile != "Localhost").map(profile, {"type": profile})
          - name: inputLocalHostProfiles
            expression: |
              variables.params.allowedProfiles.exists(profile, profile == "Localhost") ? variables.params.allowedLocalhostFiles.map(file, {"type": "Localhost", "localHostProfile": string(file)}) : []
          - name: inputAllowedProfiles
            expression: |
              variables.inputNonLocalHostProfiles + variables.inputLocalHostProfiles
          - name: hasPodSeccomp
            expression: |
              has(variables.anyObject.spec.securityContext) && has(variables.anyObject.spec.securityContext.seccompProfile)
          - name: podLocalHostProfile
            expression: |
              variables.hasPodSeccomp && has(variables.anyObject.spec.securityContext.seccompProfile.localhostProfile) ? variables.anyObject.spec.securityContext.seccompProfile.localhostProfile : ""
          - name: podSecurityContextProfileType
            expression: |
              has(variables.hasPodSeccomp) && has(variables.anyObject.spec.securityContext.seccompProfile.type) ? variables.anyObject.spec.securityContext.seccompProfile.type
                : ""
          - name: podSecurityContextProfiles
            expression: |
              variables.unverifiedContainers.filter(container, 
                !(has(container.securityContext) && has(container.securityContext.seccompProfile)) && 
                variables.hasPodSeccomp
              ).map(container, {
                "container" : container.name,
                "profile" : dyn(variables.podSecurityContextProfileType),
                "file" : variables.podLocalHostProfile,
                "location" : dyn("pod securityContext"),
              })
          - name: containerSecurityContextProfiles
            expression: |
              variables.unverifiedContainers.filter(container, 
                has(container.securityContext) && has(container.securityContext.seccompProfile)
              ).map(container, {
                "container" : container.name,
                "profile" : dyn(container.securityContext.seccompProfile.type),
                "file" : has(container.securityContext.seccompProfile.localhostProfile) ? container.securityContext.seccompProfile.localhostProfile : dyn(""),
                "location" : dyn("container securityContext"),
              })
          - name: containerProfilesMissing
            expression: |
              variables.unverifiedContainers.filter(container, 
                !(has(container.securityContext) && has(container.securityContext.seccompProfile)) && 
                !variables.hasPodSeccomp
              ).map(container, {
                "container" : container.name,
                "profile" : dyn("not configured"),
                "file" : dyn(""),
                "location" : dyn("no explicit profile found"),
              })
          - name: allContainerProfiles
            expression: |
              variables.podSecurityContextProfiles + variables.containerSecurityContextProfiles + variables.containerProfilesMissing
          - name: badContainerProfilesWithoutFiles
            expression: |
              variables.allContainerProfiles.filter(container, 
                  container.profile != "Localhost" &&
                  !variables.inputAllowedProfiles.exists(profile, profile.type == container.profile)
              ).map(badProfile, "Seccomp profile '" + badProfile.profile + "' is not allowed for container '" + badProfile.container + "'. Found at: " + badProfile.location + ". Allowed profiles: " + variables.inputAllowedProfiles.map(profile, "{\"type\": \"" + profile.type + "\"" + (has(profile.localHostProfile) ? ", \"localHostProfile\": \"" + profile.localHostProfile + "\"}" : "}")).join(", "))
          - name: badContainerProfilesWithFiles
            expression: |
              variables.allContainerProfiles.filter(container, 
                container.profile == "Localhost" &&
                !variables.inputAllowedProfiles.exists(profile, profile.type == "Localhost" && (has(profile.localHostProfile) && (profile.localHostProfile == container.file || profile.localHostProfile == "*")))
              ).map(badProfile, "Seccomp profile '" + badProfile.profile + "' With file '" + badProfile.file + "' is not allowed for container '" + badProfile.container + "'. Found at: " + badProfile.location + ". Allowed profiles: " + variables.inputAllowedProfiles.map(profile, "{\"type\": \"" + profile.type + "\"" + (has(profile.localHostProfile) ? ", \"localHostProfile\": \"" + profile.localHostProfile + "\"}" : "}")).join(", "))
          validations:
          - expression: 'size(variables.badContainerProfilesWithoutFiles) == 0'
            messageExpression: |
              variables.badContainerProfilesWithoutFiles.join(", ")
          - expression: 'size(variables.badContainerProfilesWithFiles) == 0'
            messageExpression: |
              variables.badContainerProfilesWithFiles.join(", ")
      - engine: Rego
        source:
          rego: |
            package k8spspseccomp

            import data.lib.exempt_container.is_exempt

            violation[{"msg": msg}] {
                not input_wildcard_allowed_profiles
                allowed_profiles := get_allowed_profiles
                container := input_containers[name]
                not is_exempt(container)
                result := get_profile(container)
                not allowed_profile(result.profile, result.file, allowed_profiles)
                msg := get_message(result.profile, result.file, name, result.location, allowed_profiles)
            }

            get_message(profile, _, name, location, allowed_profiles) = message {
                profile != "Localhost"
                message := sprintf("Seccomp profile '%v' is not allowed for container '%v'. Found at: %v. Allowed profiles: %v", [profile, name, location, allowed_profiles])
            }

            get_message(profile, file, name, location, allowed_profiles) = message {
                profile == "Localhost"
                message := sprintf("Seccomp profile '%v' with file '%v' is not allowed for container '%v'. Found at: %v. Allowed profiles: %v", [profile, file, name, location, allowed_profiles])
            }

            input_wildcard_allowed_profiles {
                input.parameters.allowedProfiles[_] == "*"
            }

            input_wildcard_allowed_files {
                input.parameters.allowedLocalhostFiles[_] == "*"
            }

            allowed_profile(_, _, _) {
                input_wildcard_allowed_profiles
            }

            allowed_profile(profile, _, _) {
                profile == "Localhost"
                input_wildcard_allowed_files
            }

            # Simple allowed Profiles
            allowed_profile(profile, _, allowed) {
                profile != "Localhost"
                allow_profile = allowed[_]
                profile == allow_profile.type
            }

            # annotation localhost without wildcard
            allowed_profile(profile, file, allowed) {
                profile == "Localhost"
                allow_profile = allowed[_]
                allow_profile.type == "Localhost"
                file == allow_profile.localHostProfile
            }

            # The profiles explicitly in the list
            get_allowed_profiles[allowed] {
                profile := input.parameters.allowedProfiles[_]
                profile != "Localhost"
                allowed := {"type": profile}
            }

            get_allowed_profiles[allowed] {
                profile := input.parameters.allowedProfiles[_]
                profile == "Localhost"
                file := object.get(input.parameters, "allowedLocalhostFiles", [""])[_]
                allowed := {"type": "Localhost", "localHostProfile": file}
            }

            # Container profile as defined in containers securityContext
            get_profile(container) = {"profile": profile, "file": file, "location": location} {
                has_securitycontext_container(container)
                profile := container.securityContext.seccompProfile.type
                file := object.get(container.securityContext.seccompProfile, "localhostProfile", "")
                location := "container securityContext"
            }

            # Container profile as defined in pods securityContext
            get_profile(container) = {"profile": profile, "file": file, "location": location} {
                not has_securitycontext_container(container)
                profile := input.review.object.spec.securityContext.seccompProfile.type
                file := object.get(input.review.object.spec.securityContext.seccompProfile, "localhostProfile", "")
                location := "pod securityContext"
            }

            # Container profile missing
            get_profile(container) = {"profile": "not configured", "file": "", "location": "no explicit profile found"} {
                not has_securitycontext_container(container)
                not has_securitycontext_pod
            }

            has_securitycontext_pod {
                input.review.object.spec.securityContext.seccompProfile
            }

            has_securitycontext_container(container) {
                container.securityContext.seccompProfile
            }

            input_containers[container.name] = container {
                container := input.review.object.spec.containers[_]
            }

            input_containers[container.name] = container {
                container := input.review.object.spec.initContainers[_]
            }

            input_containers[container.name] = container {
                container := input.review.object.spec.ephemeralContainers[_]
            }
          libs:
            - |
              package lib.exempt_container

              is_exempt(container) {
                  exempt_images := object.get(object.get(input, "parameters", {}), "exemptImages", [])
                  img := container.image
                  exemption := exempt_images[_]
                  _matches_exemption(img, exemption)
              }

              _matches_exemption(img, exemption) {
                  not endswith(exemption, "*")
                  exemption == img
              }

              _matches_exemption(img, exemption) {
                  endswith(exemption, "*")
                  prefix := trim_suffix(exemption, "*")
                  startswith(img, prefix)
              }

```

### Usage
```shell
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/pod-security-policy/seccompv2/template.yaml
```
## Examples
<details>
<summary>default-seccomp-required</summary>

<details>
<summary>constraint</summary>

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPSeccompV2
metadata:
  name: psp-seccomp
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    exemptImages:
    - nginx-exempt 
    allowedProfiles:
    - RuntimeDefault
    - Localhost
    allowedLocalhostFiles:
    - "*"

```

Usage

```shell
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/pod-security-policy/seccompv2/samples/psp-seccomp/constraint.yaml
```

</details>

<details>
<summary>example-disallowed-global</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-seccomp-disallowed2
  labels:
    app: nginx-seccomp
spec:
  securityContext:
    seccompProfile:
      type: Unconfined
  containers:
  - name: nginx
    image: nginx

```

Usage

```shell
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/pod-security-policy/seccompv2/samples/psp-seccomp/example_disallowed2.yaml
```

</details>
<details>
<summary>example-disallowed-container</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-seccomp-disallowed
  labels:
    app: nginx-seccomp
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      seccompProfile:
        type: Unconfined

```

Usage

```shell
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/pod-security-policy/seccompv2/samples/psp-seccomp/example_disallowed.yaml
```

</details>
<details>
<summary>example-allowed-container</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-seccomp-allowed
  labels:
    app: nginx-seccomp
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      seccompProfile:
        type: RuntimeDefault

```

Usage

```shell
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/pod-security-policy/seccompv2/samples/psp-seccomp/example_allowed.yaml
```

</details>
<details>
<summary>example-allowed-container</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-seccomp-allowed-localhost
  labels:
    app: nginx-seccomp
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      seccompProfile:
        type: Localhost
        localhostProfile: profile.json

```

Usage

```shell
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/pod-security-policy/seccompv2/samples/psp-seccomp/example_allowed_localhost.yaml
```

</details>
<details>
<summary>example-allowed-container-exempt-image</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-seccomp-disallowed
  labels:
    app: nginx-seccomp
spec:
  containers:
  - name: nginx
    image: nginx-exempt
    securityContext:
      seccompProfile:
        type: Unconfined

```

Usage

```shell
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/pod-security-policy/seccompv2/samples/psp-seccomp/example_allowed_exempt_image.yaml
```

</details>
<details>
<summary>disallowed-ephemeral</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-seccomp-disallowed
  labels:
    app: nginx-seccomp
spec:
  ephemeralContainers:
  - name: nginx
    image: nginx

```

Usage

```shell
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/pod-security-policy/seccompv2/samples/psp-seccomp/disallowed_ephemeral.yaml
```

</details>


</details>
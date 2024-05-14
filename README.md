# Custom Renovate configuration preset

This is my custom Renovate configuration preset :wave:


## Google Cloud Artifact Registry

To pull images from a private Google Cloud Artifact Registry, I use something like this (using Terraform):

```hcl
resource "google_service_account" "renovate" {
  account_id   = "renovate"
  display_name = "Renovate"
}

resource "google_artifact_registry_repository_iam_member" "renovate" {
  role       = "roles/artifactregistry.reader"
  location   = google_artifact_registry_repository.this.location
  repository = google_artifact_registry_repository.this.name
  member     = "serviceAccount:${google_service_account.renovate.email}"
}

resource "google_service_account_key" "renovate" {
  service_account_id = google_service_account.renovate.name
}

output "renovate_private_key" {
  sensitive = true
  value     = base64encode(jsonencode(jsondecode(base64decode(nonsensitive(google_service_account_key.renovate.private_key)))))
EOF
}
```

This produces a sensitive output containing the private key. To use this key wih Renovate:

* Encrypt the key with https://app.renovatebot.com/encrypt:
  * Set the `Organization` field to your GitHub organiation (or personal account)
  * Encrypt the key
* Configure Renovate with something like this:
  ```json5
  "hostRules": [
    {
      "hostType": "docker",
      "matchHost": "europe-west3-docker.pkg.dev",
      "username": "_json_key_base64",
      "encrypted": {
        "password": "encrypted content from https://app.renovatebot.com/encrypt",
      },
    },
  ]
  ```

> [!IMPORTANT]
>
> Renovate caches results really aggressively: if Renovate is not authorized to find images in the registry, it will still cache "no result" for some time (at least one hour).
>
> This can be really confusing: even if the authentication and configuration is correctly set, Renovate may still display that it's not able to find new updates.
>
> This will eventually work after some time, once the cache expires and Renovate *really* tries to fetch new information from the Docker registry.
>
> A hack to force Renovate to find new versions and test the configuration is to change the image name to be discovered:
>
> * Push a new image in your registry, for instance: `foobar2:v1.1.0` (instead of `foobar:v1.1.0`)
> * Change your source code to pull the image `foobar:v1.0.0` instead of `foobar2:v1.0.0`
> * Renovate "doesn't know" yet about `foobar2` and will try to populate its cache, reusing then the new authentication configuration.
>
> Good luck!

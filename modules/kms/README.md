# Google KMS Module

This module allows creating and managing KMS crypto keys and IAM bindings at both the keyring and crypto key level. An existing keyring can be used, or a new one can be created and managed by the module if needed.

When using an existing keyring be mindful about applying IAM bindings, as all bindings used by this module are authoritative, and you might inadvertently override bindings managed by the keyring creator.

<!-- BEGIN TOC -->
- [Protecting against destroy](#protecting-against-destroy)
- [Examples](#examples)
  - [Using an existing keyring](#using-an-existing-keyring)
  - [Keyring creation and crypto key rotation and IAM roles](#keyring-creation-and-crypto-key-rotation-and-iam-roles)
  - [Crypto key purpose](#crypto-key-purpose)
- [Variables](#variables)
- [Outputs](#outputs)
<!-- END TOC -->

## Protecting against destroy

In this module **no lifecycle blocks are set on resources to prevent destroy**, in order to allow for experimentation and testing where rapid `apply`/`destroy` cycles are needed. If you plan on using this module to manage non-development resources, **clone it and uncomment the lifecycle blocks** found in `main.tf`.

## Examples

### Using an existing keyring

```hcl
module "kms" {
  source     = "./fabric/modules/kms"
  project_id = "my-project"
  iam = {
    "roles/cloudkms.admin" = ["user:user1@example.com"]
  }
  keyring        = { location = "europe-west1", name = "test" }
  keyring_create = false
  keys           = { key-a = null, key-b = null, key-c = null }
}
# tftest skip (uses data sources)
```

### Keyring creation and crypto key rotation and IAM roles

```hcl
module "kms" {
  source     = "./fabric/modules/kms"
  project_id = "my-project"
  iam_additive = {
    "roles/cloudkms.cryptoKeyEncrypterDecrypter" = [
      "user:user1@example.com", "user:user2@example.com"
    ]
  }
  key_iam = {
    key-a = {
      "roles/cloudkms.admin" = ["user:user3@example.com"]
    }
  }
  key_iam_additive = {
    key-b = {
      "roles/cloudkms.cryptoKeyEncrypterDecrypter" = [
        "user:user4@example.com", "user:user5@example.com"
      ]
    }
  }
  key_iam_members = {
    key-b-am1 = {
      key    = "key-b"
      member = "user:am1@example.com"
      role   = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
    }
  }
  keyring = { location = "europe-west1", name = "test" }
  keys = {
    key-a = null
    key-b = { rotation_period = "604800s", labels = null }
    key-c = { rotation_period = null, labels = { env = "test" } }
  }
}
# tftest modules=1 resources=10
```

### Crypto key purpose

```hcl
module "kms" {
  source     = "./fabric/modules/kms"
  project_id = "my-project"
  key_purpose = {
    key-c = {
      purpose = "ASYMMETRIC_SIGN"
      version_template = {
        algorithm        = "EC_SIGN_P384_SHA384"
        protection_level = null
      }
    }
  }
  keyring = { location = "europe-west1", name = "test" }
  keys    = { key-a = null, key-b = null, key-c = null }
}
# tftest modules=1 resources=4
```
<!-- BEGIN TFDOC -->
## Variables

| name | description | type | required | default |
|---|---|:---:|:---:|:---:|
| [keyring](variables.tf#L91) | Keyring attributes. | <code title="object&#40;&#123;&#10;  location &#61; string&#10;  name     &#61; string&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> | ✓ |  |
| [project_id](variables.tf#L114) | Project id where the keyring will be created. | <code>string</code> | ✓ |  |
| [iam](variables.tf#L17) | Keyring IAM bindings in {ROLE => [MEMBERS]} format. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [iam_additive](variables.tf#L23) | Keyring IAM additive bindings in {ROLE => [MEMBERS]} format. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [iam_members](variables.tf#L29) | Individual additive IAM bindings, use this when iam_additive does not work due to dynamic resources. Keys are arbitrary and only used for the internal loop. | <code title="map&#40;object&#40;&#123;&#10;  member &#61; string&#10;  role   &#61; string&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [key_iam](variables.tf#L39) | Key IAM bindings in {KEY => {ROLE => [MEMBERS]}} format. | <code>map&#40;map&#40;list&#40;string&#41;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [key_iam_additive](variables.tf#L45) | Key IAM additive bindings in {KEY => {ROLE => [MEMBERS]}} format. | <code>map&#40;map&#40;list&#40;string&#41;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [key_iam_members](variables.tf#L51) | Individual additive IAM bindings, use this when iam_additive does not work due to dynamic resources. Keys are arbitrary and only used for the internal loop. | <code title="map&#40;object&#40;&#123;&#10;  key    &#61; string&#10;  member &#61; string&#10;  role   &#61; string&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [key_purpose](variables.tf#L62) | Per-key purpose, if not set defaults will be used. If purpose is not `ENCRYPT_DECRYPT` (the default), `version_template.algorithm` is required. | <code title="map&#40;object&#40;&#123;&#10;  purpose &#61; string&#10;  version_template &#61; object&#40;&#123;&#10;    algorithm        &#61; string&#10;    protection_level &#61; string&#10;  &#125;&#41;&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [key_purpose_defaults](variables.tf#L74) | Defaults used for key purpose when not defined at the key level. If purpose is not `ENCRYPT_DECRYPT` (the default), `version_template.algorithm` is required. | <code title="object&#40;&#123;&#10;  purpose &#61; string&#10;  version_template &#61; object&#40;&#123;&#10;    algorithm        &#61; string&#10;    protection_level &#61; string&#10;  &#125;&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code title="&#123;&#10;  purpose          &#61; null&#10;  version_template &#61; null&#10;&#125;">&#123;&#8230;&#125;</code> |
| [keyring_create](variables.tf#L99) | Set to false to manage keys and IAM bindings in an existing keyring. | <code>bool</code> |  | <code>true</code> |
| [keys](variables.tf#L105) | Key names and base attributes. Set attributes to null if not needed. | <code title="map&#40;object&#40;&#123;&#10;  rotation_period &#61; string&#10;  labels          &#61; map&#40;string&#41;&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [tag_bindings](variables.tf#L119) | Tag bindings for this keyring, in key => tag value id format. | <code>map&#40;string&#41;</code> |  | <code>null</code> |

## Outputs

| name | description | sensitive |
|---|---|:---:|
| [id](outputs.tf#L17) | Fully qualified keyring id. |  |
| [key_ids](outputs.tf#L25) | Fully qualified key ids. |  |
| [keyring](outputs.tf#L36) | Keyring resource. |  |
| [keys](outputs.tf#L44) | Key resources. |  |
| [location](outputs.tf#L52) | Keyring location. |  |
| [name](outputs.tf#L60) | Keyring name. |  |
<!-- END TFDOC -->

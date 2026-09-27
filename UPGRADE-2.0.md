# Upgrade from v1.x to v2.x

Please consult the `examples` directory for reference example configurations. If you find a bug, please open an issue with supporting configuration to reproduce.

## List of backwards incompatible changes

- Minimum supported version of Terraform AWS provider updated to v4.21 to support latest resources
- Minimum supported version of Terraform updated to v1.0
- The variable `create_key_pair` is now simply `create`

## Additional changes

### Added

- Support for creating private key within the module using the commonly used `tls_private_key` resource

### Modified

  - None

### Removed

  - None

### Variable and output changes

1. Removed variables:

  - None

2. Renamed variables:

  - `create_key_pair` -> `create`

3. Added variables:

  - `create_private_key`
  - `private_key_algorithm`
  - `private_key_rsa_bits`

4. Removed outputs:

    - None

5. Renamed outputs:

    - `key_pair_key_pair_id` -> `key_pair_id`
    - `key_pair_key_name` -> `key_pair_name`


6. Added outputs:

    - `key_pair_arn`
    - `private_key_id`
    - `private_key_openssh`
    - `private_key_pem`
    - `public_key_fingerprint_md5`
    - `public_key_fingerprint_sha256`
    - `public_key_openssh`
    - `public_key_pem`

## Upgrade Migrations

### State Move Commands

None required


##############################################
# LOOSE FILE MOVE FUNCTION APP
# One Windows Function app per environment/region stack, shared by all pods.
##############################################

##############################################
# APP SERVICE PLAN (WINDOWS, DEDICATED)
##############################################

resource "azurerm_service_plan" "loose_file_move" {
  count = var.loose_file_move_enabled ? 1 : 0

  name                = local.loose_file_move_names.service_plan
  resource_group_name = azurerm_resource_group.mft_rg.name
  location            = var.location
  os_type             = "Windows"
  sku_name            = var.function_service_plan_sku

  tags = merge(local.base_tags, {})
}

##############################################
# RUNTIME STORAGE ACCOUNT (FUNCTIONS HOST)
##############################################

resource "azurerm_storage_account" "loose_file_move_runtime" {
  count = var.loose_file_move_enabled ? 1 : 0

  name                = local.loose_file_move_names.storage_account
  resource_group_name = azurerm_resource_group.mft_rg.name
  location            = var.location

  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"

  min_tls_version                 = "TLS1_2"
  shared_access_key_enabled       = false
  allow_nested_items_to_be_public = false
  public_network_access           = "Enabled"

  tags = merge(local.base_tags, {})
}

resource "azurerm_storage_account_network_rules" "loose_file_move_runtime" {
  count = var.loose_file_move_enabled ? 1 : 0

  storage_account_id = azurerm_storage_account.loose_file_move_runtime[0].id
  default_action     = "Deny"
  bypass             = ["AzureServices"]
  ip_rules           = var.function_runtime_storage_allowed_ip_rules
}

resource "azurerm_private_endpoint" "loose_file_move_runtime" {
  for_each = var.loose_file_move_enabled ? toset(["blob", "queue", "table"]) : toset([])

  name                = "${azurerm_storage_account.loose_file_move_runtime[0].name}-${each.key}-pe"
  resource_group_name = azurerm_resource_group.mft_rg.name
  location            = var.location
  subnet_id           = var.private_endpoint_subnet_id

  private_service_connection {
    name                           = "${azurerm_storage_account.loose_file_move_runtime[0].name}-${each.key}-psc"
    private_connection_resource_id = azurerm_storage_account.loose_file_move_runtime[0].id
    subresource_names              = [each.key]
    is_manual_connection           = false
  }

  dynamic "private_dns_zone_group" {
    for_each = contains(keys(var.private_dns_zone_ids), each.key) ? [1] : []
    content {
      name                 = "default"
      private_dns_zone_ids = [var.private_dns_zone_ids[each.key]]
    }
  }

  tags = merge(local.base_tags, {})
}

##############################################
# APPLICATION INSIGHTS
##############################################

resource "azurerm_application_insights" "loose_file_move" {
  count = var.loose_file_move_enabled ? 1 : 0

  name                = local.loose_file_move_names.app_insights
  resource_group_name = azurerm_resource_group.mft_rg.name
  location            = var.location
  workspace_id        = var.log_analytics_workspace_id
  application_type    = "web"

  tags = merge(local.base_tags, {})
}

##############################################
# FUNCTION APP
##############################################

resource "azurerm_windows_function_app" "loose_file_move" {
  count = var.loose_file_move_enabled ? 1 : 0

  name                = local.loose_file_move_names.function_app
  resource_group_name = azurerm_resource_group.mft_rg.name
  location            = var.location
  service_plan_id     = azurerm_service_plan.loose_file_move[0].id

  # Host storage via the app's system-assigned identity (no keys).
  storage_account_name          = azurerm_storage_account.loose_file_move_runtime[0].name
  storage_uses_managed_identity = true

  functions_extension_version = "~4"
  https_only                  = true
  builtin_logging_enabled     = false

  # Event Grid and the Spacelift zip deploy both reach the app over its
  # public endpoint. Outbound traffic goes through VNet integration.
  public_network_access_enabled = true
  virtual_network_subnet_id     = var.function_integration_subnet_id

  # zip_deploy_file publishes through the SCM endpoint with publishing credentials.
  ftp_publish_basic_authentication_enabled       = false
  webdeploy_publish_basic_authentication_enabled = true

  zip_deploy_file = "${path.root}/packages/loosefilemove-${var.loose_file_move_version}.zip"

  identity {
    type = "SystemAssigned"
  }

  site_config {
    always_on              = true
    vnet_route_all_enabled = true
    minimum_tls_version    = "1.2"
    ftps_state             = "Disabled"

    application_insights_connection_string = azurerm_application_insights.loose_file_move[0].connection_string

    application_stack {
      dotnet_version              = var.function_dotnet_version
      use_dotnet_isolated_runtime = var.function_use_isolated_runtime
    }
  }

  app_settings = {
    WEBSITE_RUN_FROM_PACKAGE = "1"
  }

  tags = merge(local.base_tags, {})

  depends_on = [
    azurerm_private_endpoint.loose_file_move_runtime,
    azurerm_storage_account_network_rules.loose_file_move_runtime,
  ]
}

##############################################
# RBAC - FUNCTION APP SYSTEM-ASSIGNED IDENTITY
##############################################

# Functions host storage: leases, keys, internal queues and tables.
resource "azurerm_role_assignment" "loose_file_move_runtime_storage" {
  for_each = var.loose_file_move_enabled ? toset([
    "Storage Blob Data Owner",
    "Storage Queue Data Contributor",
    "Storage Table Data Contributor",
  ]) : toset([])

  scope                = azurerm_storage_account.loose_file_move_runtime[0].id
  role_definition_name = each.key
  principal_id         = azurerm_windows_function_app.loose_file_move[0].identity[0].principal_id
}

# Pod storage accounts: move loose files.
resource "azurerm_role_assignment" "loose_file_move_pod_blob" {
  for_each = var.loose_file_move_enabled ? var.storage_accounts : {}

  scope                = azurerm_storage_account.this[each.key].id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = azurerm_windows_function_app.loose_file_move[0].identity[0].principal_id
}

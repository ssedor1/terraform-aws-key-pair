# Examples

Please note - the examples provided serve two primary means:

1. Show users working examples of the various ways in which the module can be configured and features supported
2. A means of testing/validating module changes

Please do not mistake the examples provided as "best practices". It is up to users to consult the AWS service documentation for best practices, usage recommendations, etc.
locals {
  # Add to each of the four existing entries:
  #   endpoint_type = "storage_queue"

  loose_file_move_subscriptions = {
    loose_blob_created_move = {
      name          = "loose-blob-created-move"
      event_type    = "Microsoft.Storage.BlobCreated"
      endpoint_type = "azure_function"
      queue_name    = null
      filter_key    = "data.blobUrl"
      operator      = "not_ends_with"
      extensions    = concat(local.archive_extensions, [".filepart"])
      exclude_paths = ["/Outgoing/"]
      api_filter    = local.created_api_filter
    }

    loose_blob_renamed_move = {
      name          = "loose-blob-renamed-move"
      event_type    = "Microsoft.Storage.BlobRenamed"
      endpoint_type = "azure_function"
      queue_name    = null
      filter_key    = "data.destinationUrl"
      operator      = "not_ends_with"
      extensions    = local.archive_extensions
      exclude_paths = ["/Outgoing/"]
      api_filter    = []
    }
  }

  all_event_subscriptions = merge(
    local.event_subscriptions,
    var.loose_file_move_enabled && var.loose_file_move_subscriptions_enabled ? local.loose_file_move_subscriptions : {}
  )

  event_subscription_instances = merge([
    for account_key, account in var.storage_accounts : {
      for sub_key, sub in local.all_event_subscriptions :
      "${account_key}/${sub_key}" => merge(sub, {
        storage_account_key = account_key
        queue_key           = sub.queue_name != null ? "${account_key}/${sub.queue_name}" : null
      })
    }
  ]...)

  loose_file_move_function_id = (
    var.loose_file_move_enabled
    ? "${azurerm_windows_function_app.loose_file_move[0].id}/functions/LooseFileMove"
    : null
  )
}





resource "azurerm_eventgrid_system_topic_event_subscription" "this" {
  for_each = local.event_subscription_instances

  name                 = each.value.name
  system_topic         = azurerm_eventgrid_system_topic.storage[each.value.storage_account_key].name
  resource_group_name  = azurerm_resource_group.mft_rg.name
  included_event_types = [each.value.event_type]

  # Managed identity delivery applies to queue endpoints only.
  dynamic "delivery_identity" {
    for_each = each.value.endpoint_type == "storage_queue" ? [1] : []
    content {
      type = "SystemAssigned"
    }
  }

  dynamic "storage_queue_endpoint" {
    for_each = each.value.endpoint_type == "storage_queue" ? [1] : []
    content {
      storage_account_id = azurerm_storage_account.this[each.value.storage_account_key].id
      queue_name         = azurerm_storage_queue.this[each.value.queue_key].name
    }
  }

  dynamic "azure_function_endpoint" {
    for_each = each.value.endpoint_type == "azure_function" ? [1] : []
    content {
      function_id                       = local.loose_file_move_function_id
      max_events_per_batch              = 1
      preferred_batch_size_in_kilobytes = 64
    }
  }

  retry_policy {
    max_delivery_attempts = 30
    event_time_to_live    = 1440
  }

  # dynamic "advanced_filter" { ... unchanged ... }

  depends_on = [
    time_sleep.eventgrid_rbac_propagation,
    azurerm_windows_function_app.loose_file_move,
  ]
}

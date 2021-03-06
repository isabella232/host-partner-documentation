# Queue a Restore

Perform a restore on the given site to the specified timestamp.

### Endpoint Information

- __Method__: POST
- __URL__: `https://public-api.wordpress.com/wpcom/v1.1/activity-log/<blog_id>/rewind/to/<timestamp>`
- __blog_id__: (number) Site to restore.
- __timestamp__: (number) Unix timestamp to restore to.

Authentication: [Access token](/jetpack/jetpack-start-endpoints/authentication.md).

### Request Parameters

- __roleName__: (string) The credential role to restore to, one of `main|alternate|import`.
- __types__: (optional string / array) Data and file types to restore, one or more of `all|themes|plugins|uploads|sqls|contents|roots`. Defaults to `all`.
- __destinationSiteName__: (optional string) Title to give the site when using the `alternate` role.

### Successful response

```
{
  "ok":true,
  "error":"",
  "restore_id":234725,
  "job_id":4661184823
}
```

### Example

Restore site 155308227 to 10/29/2020 12:47pm UTC:

`https://public-api.wordpress.com/wpcom/v1.1/activity-log/155308227/rewind/to/1603975624`

### Notes

The restore will use the closest full daily snapshot not newer than the given timestamp, and (where available) append more recent incremental updates not newer than the given timestamp PLUS any immutable incremental updates (such as WooCommerce orders) up to the current time.

For a regular restore use the 'main' `roleName`. 'alternate' is for cloning to a different site and 'import' is reserved for internal use.

Restore types:
- `all`: Everything
- `themes`: Themes folder
- `plugins`: Plugins folder
- `uploads`: Uploads folder
- `sqls`: Database tables
- `contents`: wp-content folder
- `roots`: WordPress install root folder


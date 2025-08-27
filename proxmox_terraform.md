Чтобы в proxmox provider terraform получуть всю схему ресурсов и сохранить в schema.json
```
terraform providers schema -json > schema.json
```

Чтобы вывести иерархично  schema.json, чтобы понимать какой ресурс имеет какие настойки;
```
jq -r '
.provider_schemas["registry.terraform.io/telmate/proxmox"].resource_schemas
| to_entries[]
| "\(.key):"
, "  attributes: \(.value.block.attributes // {} | keys)"
, (
    (.value.block.block_types // {} | to_entries[]
     | "  block \(.key): attributes: \(.value.block.attributes // {} | keys)"
    )
  )
' schema.json
```




terraform fmt - to format config files in canonical format

terraform validate - to validate config file, if there is an issue then terraform shows line and the file that is causing issue.
What terraform validate does:
    Checks syntax correctness.
    Ensures resource references are valid within the configuration.
    Validates provider block formatting.
    Ensures required arguments are provided (based on static analysis).
    Catches issues like:
        Misspelled resource types.
        Incorrect variable usage.
        Malformed expressions.

terraform show - it show current state of infra as seen by terraform, so all resources and all their attributes.
terraform show list (to list resources)

terraform providers - to see all the used providers.
we can copy these providers to another dir using mirror command - 
terraform providers mirror  /root/terraform/new-project-dir, Maintains the correct Terraform registry structure (unlike cp).

terrform output - to print all the output from config file
to pick specific output and not all - terraform output pet-name (it will show only pet-name output)

terraform plan - to sync terraform state file with real infra. This only modify state file and not the actual infra.
- Syncs with the real infrastructure – It checks what already exists in your cloud environment.
- Compares with your Terraform configuration files – It looks at the desired state defined in your .tf files.
Shows a detailed execution plan – It displays:
- Resources already present (no changes needed).
- Resources that will be created (+ sign).
- Resources that will be updated (~ sign).
- Resources that will be destroyed (- sign).

terraform graph - to see all depenency of resources.
first install graphwiz then this command will show clear visiuallization. 
then use - 
terraform graph | dot -Tsvg > graph.svg
graph.svg in the browser and see this graph.

............................................






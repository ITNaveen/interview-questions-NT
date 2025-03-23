When something gpes wrong then use debugging and logging is the best way, we have 5 types of logging.
1. INFO
2. WARNING
3. ERROR
4. DEBUG
5. TRACE

TRACE   : ████████████████████ (Most Detailed)


DEBUG   : ████████████ 


INFO    : ████████


WARNING : ███


ERROR   : █ (Least Detailed)

- Debug and Trace is most commanly used.
In short:
✔ Use TF_LOG=DEBUG for standard debugging.
✔ Use TF_LOG=TRACE if you need deep insight into Terraform’s internal processing (e.g., for Terraform developers or extreme debugging cases).

ERROR → Logs only critical errors.
WARN → Logs warnings and errors.
INFO → Logs general operational information.

DEBUG → Logs detailed debugging information - DEBUG logs will show which resource starts, how Terraform evaluates dependencies, and how resources interact as per your configuration.

TRACE → 
Logs extremely detailed internal debugging -

- If you set TF_LOG=TRACE, you’ll see everything in DEBUG.
- Low-level Terraform internals (e.g., function calls, internal state transitions).
- xtremely detailed information about how Terraform processes resources.
- Deep-dive into the Terraform provider interactions.

# To use - 
# Set log type and path before Terraform commands
export TF_LOG=TRACE
export TF_LOG_PATH=/tmp/terraform.log

# Run Terraform commands in sequence
terraform init
terraform validate
terraform plan
terraform apply

# After all operations are complete, unset log path
unset TF_LOG_PATH

# Optional: Also unset log level if you want
unset TF_LOG


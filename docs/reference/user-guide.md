# User guide fragments

Tenant-facing language, ready to lift into a deployment's user guide.

- **The half-screen doctrine:** *The management window is for managing; your local browser is for searching.* When troubleshooting, put your Guacamole session on one half of your screen and your own browser on the other. Copy error text out of the session freely — that's what it's for.
- **Where your stuff lives:** your jump VM is rebuilt monthly. Anything you want to keep — playbooks, inventories, notes — belongs in your group's Forgejo org, which is permanent, versioned, and pre-configured on the VM.
- **Need a tool we don't have?** File a library request. Almost everything is approved; the process exists so the environment stays curated, not to say no.
- **Reimaging a server?** Use the iPXE ISO from the library over virtual media and chainload from the deploy server — it is dramatically faster than mounting a full installer ISO through your BMC.

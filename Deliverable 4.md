# Deliverable 4 — Container Deployment with Modern DevOps Practices

This lab deploys a stateless open-source application — [Subnets](https://github.com/davidc/subnets).

By the end of the lab you will have:
- Infrastructure defined in OpenTofu and applied repeatably from the command line
- A container image built, semantically versioned, and stored in a private registry
- A live HTTPS application with an automatically managed TLS certificate
- Logs flowing into a connected Log Analytics workspace
- Zero plaintext credentials anywhere in the deployment

---

# Toolkit

- [Subnets source code](https://github.com/davidc/subnets)
- [Azure Container Apps documentation](https://learn.microsoft.com/en-us/azure/container-apps/)
- [OpenTofu azurerm provider — Container App](https://registry.opentofu.org/providers/hashicorp/azurerm/latest/docs/resources/container_app)
- [OpenTofu azurerm provider — Container Registry](https://registry.opentofu.org/providers/hashicorp/azurerm/latest/docs/resources/container_registry)
- [Docker build reference](https://docs.docker.com/engine/reference/commandline/build/)
- [Semantic Versioning](https://semver.org/)

---

# Task 1 — Explore the Application and OverlayFS

Before deploying anything, understand what you are deploying — and what Docker is actually
doing under the hood when it builds and runs a container.

1. Refresh the Cloud Release repository to get the latest lab files:

```bash
cd ~/Cloud/ITS-4900-Cloud-Release/
git pull
cd Deliverable_4
```

2. Clone the subnets application next to the lab directory:

```bash
cd ~/Cloud
git clone https://github.com/davidc/subnets.git
```

## Task 1a — Build and Inspect the Image

3. Look at the application's Dockerfile:

```bash
cat ~/Cloud/subnets/Dockerfile
```

   Identify: what base image does it use?  What files does it copy?

4. Build the container image:

```bash
docker build ~/Cloud/subnets -t subnets:test
```

   The `:test` tag is a local throwaway label — it tells you (and anyone else looking)
   that this image is for exploration only, not a versioned artifact headed for the
   registry.  When OpenTofu builds the real image in Task 5 it will use a semantic
   version tag like `1.0.0`.  Using `:test` here keeps the two clearly separate and
   safe to delete independently.

5. Relate the build output to the content of the Dockerfile.

6. Show the build history — one row per Dockerfile instruction, with the size each
   layer added:

```bash
docker history subnets:test
```

7. The built image inherits configuration from the base image `php:5.6-apache`.
   Run the following to see what port is declared:

```bash
docker inspect subnets:test --format '{{json .Config.ExposedPorts}}'
```

   What port is declared, and why does this Dockerfile not need to repeat it?

## Task 1b — Copy-on-Write

OverlayFS presents a merged view of all layers to the running container.  When a process
inside the container writes to a file that exists in a lower (read-only) layer, the kernel
copies the entire file up to the container's private writable layer first — then writes.
This is called **copy-on-write (CoW)**.

8. Start the container.  Note that it is running on your local host, not in Azure yet:

```bash
docker run -d -p 5001:80 --name subnets-test subnets:test
```

9. Open `http://localhost:5001` and note the current look and feel of the page — the
   colors, heading text, and layout.

10. Make a visible change inside the running container:

```bash
docker exec subnets-test sed -i \
  's/<body>/<body style="background-color:#1a1a2e;color:#eee">/' \
  /var/www/html/subnets.html
```

   Refresh `http://localhost:5001`.  The page background should turn dark.

11. Ask Docker what changed inside the container's writable layer:

```bash
docker diff subnets-test
```

   The output uses three markers:

- `C` — file was copied up from a lower layer and modified
- `A` — file was added (did not exist in any lower layer)
- `D` — file was deleted

   Record: which files changed?  Were they `C` or `A`?  What does that tell you about
   where those files originally lived?

12. Confirm the original image is unchanged — stop and remove the container, then start a
    fresh one from the same image:

```bash
docker stop subnets-test && docker rm subnets-test
docker run -d -p 5001:80 --name subnets-test subnets:test
```

   Refresh `http://localhost:5001`.  Your changes are gone.  The image layers were never
   touched — only the container's writable layer was, and that layer is discarded when the
   container is removed.  **This is why containers are ephemeral by design.**

## Task 1c — Inspect the Overlay Mounts Directly

You can examine the actual filesystem paths Docker uses on the host for each layer.

13. Find the overlay mount paths for the running container:

```bash
docker inspect subnets-test --format '{{json .GraphDriver.Data}}' | python3 -m json.tool
```

   You will see four paths:
   - `LowerDir` — colon-separated list of read-only image layers (bottom to top)
   - `UpperDir` — the container's writable layer (your changes land here)
   - `MergedDir` — the unified view the container sees (all layers merged)
   - `WorkDir` — OverlayFS internal scratch space used during copy-on-write

14. List the files in the writable layer directly from the host.  Run this in WSL with sudo:

```bash
UPPER=$(docker inspect subnets-test --format '{{.GraphDriver.Data.UpperDir}}')
sudo ls -la $UPPER/var/www/html/
```

   You should see the file you modified in step 10 sitting in the upper layer — physically
   separate from the read-only lower layers that hold the original.

15. Stop and remove the test container:

```bash
docker stop subnets-test && docker rm subnets-test
```

   After removal, re-run the `ls` command from step 14.  The `UpperDir` path no longer
   exists.  The writable layer — and everything in it — was deleted with the container.

16. Note that this is a **stateless** application — it has no database, no persistent
    storage, and no secrets.  This makes it an ideal first container deployment.  You
    have just seen *why* stateless applications and containers are a natural fit: nothing
    important lives in the writable layer, so losing it on container removal is harmless.

## Task 1d — Extend the Image with RandoNet

The lab directory includes a second tool: **RandoNet**, an IPv4 random network
generator built with Ohio ECT branding.  Your job is to bundle it into the subnets
image and carry its look and feel into `subnets.html`.

The `RandoNet/` directory contains:

- `randonet.html` — the generator page
- `ect-logo.png` and `ect-logo-bottom-banner.png` — the ECT logo assets

17. Fork the subnets repository into your own GitHub account and clone your fork:

```bash
cd ~/Cloud
git clone https://github.com/<your-username>/subnets.git subnets-fork
```

18. Copy the RandoNet files into your fork.  Think about where in the subnets
    directory structure they belong so that Apache will serve them alongside the
    existing pages and the logo images are reachable from both pages.

19. The Dockerfile currently copies specific files into the image.  You will need
    to update it so the new files are included.  Review the existing `COPY`
    instructions and decide what changes are needed.

20. Apply the RandoNet color scheme to `subnets.html` — the background color and
    ECT logo placement.  The relevant values are in `randonet.html`'s `<style>` block.

21. Build your modified image tagged `:dev` and run it locally.  Confirm that:
    - `subnets.html` reflects the ECT look and feel
    - `randonet.html` is reachable and functional
    - The logo images load on both pages

   Document the `docker build` and `docker run` commands you used.

22. Confirm both images are present:

```bash
docker images subnets
```

   You should see both `subnets:test` and `subnets:dev`.  What does the size
   difference tell you about what your changes added to the image?

---

# Task 3 — Deploy with Infrastructure as Code

23. Authenticate the Azure CLI to your **ohio.edu** account if you have not already:

```bash
az login --use-device-code
```

24. Run the location selection script.  This queries your subscription for available
    Azure regions that support Container Apps and lets you choose one.  It writes your
    selection to `.env` and to `account.auto.tfvars` so that both shell commands
    and OpenTofu use the same region:

```bash
cd ~/Cloud/ITS-4900-Cloud-Release/Deliverable_4
bash select-location-containers.sh
```

25. Load the environment variables into your current shell:

```bash
source .env
echo "Location: $LOCATION"
echo "Subscription: $SUBSCRIPTION_ID"
```

---

# Task 3 — Deploy to Azure

26. Open `main.tf` and read through it before applying anything.  Notice that
    `admin_enabled = false` on the registry — the Container App pulls images using a
    Managed Identity instead of a stored password.

27. Copy and personalize the project variables file:

```bash
cd ~/Cloud/ITS-4900-Cloud-Release/Deliverable_4
cp project.auto.tfvars.example project.auto.tfvars
```

28. Edit `project.auto.tfvars` and set `acr_name` to something globally unique.
    **ACR** (Azure Container Registry) is Microsoft's private Docker image store —
    similar to Docker Hub but inside your Azure subscription.  The name must be
    globally unique across all of Azure.  Use your initials followed by `subnets` —
    for example `jsmithsubnets`:

```bash
nano project.auto.tfvars
```

29. Initialize and apply — this creates the registry, identity, and all supporting
    infrastructure:

```bash
tofu init
tofu apply
```

30. Record the outputs:

```
acr_login_server        = _______________________
app_url                 = _______________________
log_analytics_workspace = _______________________
```

31. Build your image and push it to ACR:

```bash
cd ~/Cloud/ITS-4900-Cloud-Release/Deliverable_4
bash build-push.sh 1.0.0
```

32. Deploy to the Container App:

```bash
tofu apply -var image_tag=1.0.0
```

33. Open `app_url`.  Confirm both tools are live:
    - The subnets calculator loads and correctly calculates a subnet
    - `randonet.html` is reachable at the same base URL and generates a valid network
    - The ECT logo images load on both pages

34. Confirm the infrastructure is idempotent — re-run plan with no changes:

```bash
tofu plan -var image_tag=1.0.0
```

   The plan should show `No changes.`

35. Confirm ACR admin is disabled:

```bash
az acr show \
  --name $(tofu output -raw acr_login_server | cut -d. -f1) \
  --resource-group deliverable-4 \
  --query "adminUserEnabled"
```

   The result should be `false`.

36. Open the Azure Portal, navigate to your Log Analytics workspace
    (`deliverable-4` resource group → `subnets-logs`), and run this query:

```kusto
ContainerAppConsoleLogs_CL
| where TimeGenerated > ago(1h)
| project TimeGenerated, ContainerName_s, Log_s
| order by TimeGenerated desc
| take 20
```

   You should see log entries from your running container.

---

# Task 4 — Cleanup

37. Destroy all Azure resources created by this lab:

```bash
tofu destroy
```

   Type `yes` when prompted.  Verify the resource group is gone:

```bash
az group show --name deliverable-4 2>&1 | grep -i "could not be found\|ResourceGroupNotFound" \
  && echo "Resource group deleted successfully."
```

---

# Task 5 - Modernize the Dockerfile

> Required for grad students / extra credit for undergrads

The subnets app ships with a Dockerfile that works — but it was written for PHP 5.6, which
reached end-of-life in December 2018.  In this task you will produce a hardened, production-ready
replacement.

43. Before writing anything, go to [https://hub.docker.com/_/php](https://hub.docker.com/_/php)
    and read the Tags page.  You need to choose a base image — answer these questions first:

    - What is the current stable PHP version?
    - What variants are available (`-apache`, `-fpm`, `-cli`, `-alpine`)?  Which one does
      this application need and why?
    - What is the difference between pinning `php:8.3-apache` versus `php:8-apache`?
      Which is more appropriate for a production image and why?

    Document your answers, then create an improved `Dockerfile` in your fork.
    Your version must include:

    - A modern base image — your choice, justified by the research above
    - At least three `LABEL` lines using the [OCI image annotation spec](https://specs.opencontainers.org/image-spec/annotations/)
      (`org.opencontainers.image.source`, `.description`, `.version`)
    - A `HEALTHCHECK` instruction so the container orchestrator can detect a crashed app
    - A `.dockerignore` file that excludes at least: `.git`, `*.md`, `*.sh`

44. Build your improved image and confirm it still works:

```bash
docker build ~/Cloud/subnets -t subnets:v2
docker run -d -p 5002:80 --name subnets-v2 subnets:v2
```

   Open `http://localhost:5002` and confirm the app works.  Then stop it:

```bash
docker stop subnets-v2 && docker rm subnets-v2
```

45. Inspect the labels you added:

```bash
docker inspect subnets:v2 --format '{{json .Config.Labels}}' | python3 -m json.tool
```

46. Written response: What risk does using an end-of-life base image introduce in a
    production deployment?  Name at least two categories of concern.


# Task 5 — Reflection (Written Response)

Answer the following questions in your submission:

38. The Container App pulled images from ACR without any stored username or password.
    What Azure feature makes this possible, and why is it preferable to storing credentials?

39. TLS certificates have a fixed expiry date and must be renewed periodically.
    Explain how Azure Container Apps handles certificate renewal and why this matters
    for a service that needs to stay available long-term.

40. This application has no database and no persistent storage.  What would change
    about this deployment if the application needed to store user data?  Think about
    what happens to the container's writable layer when it restarts.

41. If this deployment had been built manually through the Azure Portal with no IaC and
    no documentation, what challenges would a new engineer face six months from now when
    trying to modify or recover the service?

---

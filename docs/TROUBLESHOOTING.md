# Troubleshooting

-   When troubleshooting "permission denied" errors from `auth` for Workload
    Identity, the first step is to ask the `auth` plugin to generate an OAuth
    access token. Do this by adding `token_format: 'access_token'` to your YAML:

    ```yaml
    - uses: 'google-github-actions/auth@v0'
      with:
        # ...
        token_format: 'access_token'
    ```

    If your workflow _succeeds_ after adding the step to generate an access
    token, it means Workload Identity Federation is configured correctly and the
    issue is in subsequent actions. You can remove the `token_format` from your
    YAML. To further debug:

    1.  Look at the [debug logs][debug-logs] to see exactly which step is
        failing. Ensure you are using the latest version of that GitHub Action.

    1.  Make sure you use `actions/checkout@v2` **before** the `auth` action in
        your workflow.

    1.  If the failing action is from `google-github-action/*`, please file an
        issue in the corresponding repository.

    1.  If the failing action is from an external action, please file an issue
        against that repository. The `auth` action exports Google Application
        Default Credentials (ADC). Ask the action author to ensure they are
        processing ADC correctly and using the latest versions of the Google
        client libraries. Please note that we do not have control over actions
        outside of `google-github-actions`.

    If your workflow _fails_ after adding the the step to generate an access
    token, it likely means there is a misconfiguration with Workload Identity.
    Here are some common sources of errors:

    1.  Look at the [debug logs][debug-logs] to see exactly which step is
        failing. Ensure you are using the latest version of that GitHub Action.

    1.  Ensure the value for `workload_identity_provider` is the full _Provider_
        name, **not** the _Pool_ name:

        ```diff
        - projects/NUMBER/locations/global/workloadIdentityPools/POOL
        + projects/NUMBER/locations/global/workloadIdentityPools/POOL/providers/PROVIDER
        ```

    1.  Ensure you have created an **Attribute Mapping** for any **Attribute
        Conditions** or **Service Account Impersonation** principals. You cannot
        create an Attribute Condition unless you map that value from the
        incoming GitHub OIDC token. You cannot grant permissions to impersonate
        a Service Account on an attribute unless you map that value from the
        incoming GitHub OIDC token.

    1.  Ensure you have waited at least 5 minutes between making changes to the
        Workload Identity Pool and Workload Identity Provider. Changes to these
        resources are eventually consistent.

-   "The size of mapped attribute exceeds the 127 bytes limit." This error
    indicates that the GitHub OIDC token had a claim that exceeded the maximum
    allowed value of 127 bytes. In general, 1 byte = 1 character. This most
    common reason this occurs is due to long repo names or long branch names.

    **This is a limit imposed by Google Cloud IAM.** We have no control over
    this value. It is documented [here][wif-byte-limit]. Please [file feedback
    with the Google Cloud IAM team][iam-feedback]. The only mitigation is to use
    shorter repo names or shorter branch names.

-   The credentials file was bundled into my binary, container, or pull request!
    By default, the `auth` action exports credentials to the current workspace
    so that the credentials are automatically available to future steps and
    Docker-based actions. The credentials file is automatically removed when the
    job finishes.

    This means, after `auth` completes, the workspace is dirty and contains a
    credentials file. This means creating a pull request, compiling a binary, or
    building a Docker container, will include said credential file. There are a
    few ways to fix this issue:

    -   Re-order your steps. In most cases, you can re-order your steps such
        that `auth` comes _after_ the "compilation" step:

        ```text
        1. Checkout
        2. Compile (e.g. "docker build", "go build", "git add")
        3. Auth
        4. Push
        ```

        This ensures that no authentication data is present during artifact
        creation.

    -   In situations where `auth` must occur before compilation, you can use
        the output to exclude the credential:

        ```text
        1. Checkout
        2. Auth
        3. Inject "${{ steps.auth.outputs.credentials_file_path }}" into ignore file (e.g. .gitignore, .dockerignore)
        4. Compile (e.g. "docker build", "go build", "git add")
        5. Push
        ```

[debug-logs]: https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging
[iam-feedback]: https://cloud.google.com/iam/docs/getting-support
[wif-byte-limit]: https://cloud.google.com/iam/docs/configuring-workload-identity-federation

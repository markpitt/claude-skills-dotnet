# Giving consumer repositories read access

Contents

1. The problem
2. Option A: grant each package to each consumer
3. Option B: a read token in a secret
4. Choosing between them
5. Setting, checking and rotating the secret
6. Verifying a consumer restores

## 1. The problem

The `GITHUB_TOKEN` that a workflow receives is scoped to its own repository. For GitHub Packages,
that means a consumer repository's `GITHUB_TOKEN` can restore a package from another repository
only when that package has granted the consumer access. Without the grant, the restore fails with
a 401 or 403 that does not name the cause, and it looks like a credentials fault.

Two ways round it exist. Neither is free.

## 2. Option A: grant each package to each consumer

Each package has its own Actions access list. Add the consumer repository to that list on the
package's settings page: open the package, go to its settings, and use the Manage Actions access
section to add the repository with read access.

Use this when:

- there is one or two packages and one or two consumers;
- the consumers are stable and the list will not grow;
- nobody needs a token to be created or rotated.

It does not scale. The grants are per package and per consumer, so the number of grants is
packages multiplied by consumers. A new package needs every existing consumer added to it, and a
new consumer needs every package granted. Nothing in the repository's own configuration shows
the grants, so a missing one is found only when a restore fails in a repository that did not
change.

## 3. Option B: a read token in a secret

A personal access token with `read:packages` is stored as a secret on each consumer, and the
restore step passes it to NuGet through the `%VAR%` credentials in `nuget.config`
(see `feed-and-nuget-config.md`).

Points to settle before creating the token:

- **Whose account.** A token belongs to the account that created it. Use a dedicated machine
  account or a service-owned account, not a person's day-to-day login, so that the token survives
  a change of role. The account needs read access to the packages' owner.
- **Token type.** The classic personal access token with `read:packages` is the form the
  reference workflows use. Fine-grained tokens have limited support for packages at the time of
  writing, so check the current GitHub documentation before choosing one.
- **Scope.** Read only. Never give the token `write:packages` or repository write scopes for a
  consumer.
- **Expiry.** Set one, and diary the rotation.
- **Visibility of the secret.** On a free organisation, organisation-level secrets are not
  available to private repositories. Set the same secret on each consumer repository with
  `gh secret set` (section 5), ideally from a small helper that loops over the consumer list. On
  a paid plan, an organisation secret scoped to selected repositories is the cleaner route.

## 4. Choosing between them

| Situation | Choose |
|---|---|
| One package, one consumer, no change expected | Option A |
| Several packages or consumers, or consumers appear over time | Option B |
| Consumers are public repositories | Check first whether an anonymous restore works. Public packages may still need a token on GitHub Packages, so test before adding a secret. |
| Local developers restore from the feed | Use the interactive `gh` token with `read:packages` (see `feed-and-nuget-config.md` section 2) |

## 5. Setting, checking and rotating the secret

Store the token from a file or stdin. Never pass it as an argument:

```bash
gh secret set GITHUB_PACKAGES_TOKEN --repo OWNER/CONSUMER < "$TOKEN_FILE"
gh variable set GITHUB_PACKAGES_USER --repo OWNER/CONSUMER --body "LOGIN"
rm -f -- "$TOKEN_FILE"
```

`TOKEN_FILE` should be created with mode 600 and removed straight after. Do not `echo` the token
to the terminal.

Check what exists. This lists names and update times, never values:

```bash
gh secret list --repo OWNER/CONSUMER
gh variable list --repo OWNER/CONSUMER
```

Rotate in this order, so that a consumer is never left without a working token:

1. Create the new token on the service account, with the same scope.
2. Set it on each consumer with `gh secret set`, as above.
3. Run a restore in each consumer (section 6). Only then continue.
4. Revoke the old token in the account's token settings.
5. Record the rotation date wherever the team tracks expiries.

## 6. Verifying a consumer restores

Run the restore in a clean workflow run, not on a warm cache. A cached `obj/` folder can hide a
broken source:

```bash
dotnet nuget locals all --clear
dotnet restore --force
dotnet nuget list source
```

`dotnet nuget list source` should show the GitHub source as enabled. If `restore` fails with
401 or 403, check in this order: the variable and secret names match `nuget.config`; the token has
`read:packages`; the account can see the package; the consumer appears in the package's access
list (for Option A).

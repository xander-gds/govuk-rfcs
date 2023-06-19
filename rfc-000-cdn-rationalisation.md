# Rationalise our CDN config, and move away from VCL on Fastly

<!--## Summary

TODO

We currently configure our CDN (Fastly) using a domain-specific language called [VCL](https://developer.fastly.com/learning/vcl/using/).

We have CDN configuration spread over multiple repositories, which is recognised tech debt. This situation is only going to worsen if we don’t make an active effort to fix it, as we’re now introducing even more CDN config to govuk-aws in an effort to fix our Cloudfront offering so that we can ‘fail over’ in the event of a Fastly outage.

We should:

- Where possible, relocate logic from the CDN layer to other parts of the stack
- Configure equivalent but different CDN distributions on Fastly and Cloudfront, using Terraform, to handle the logic that we can’t move elsewhere
- Retire our use of VCL on Fastly

## Problems

- Currently our Fastly services and our CloudFront distributions are configured in different ways, within different codebases:
  - Our Fastly services are configured using [VCL (varnish cache language)](https://developer.fastly.com/learning/vcl/using/), in [`govuk-cdn-config`](https://github.com/alphagov/govuk-cdn-config).
  - Our CloudFront failover distribution is configured and deployed using Terraform, in [the `infra-mirror-bucket` project in `govuk-aws`](https://github.com/alphagov/govuk-aws/tree/main/terraform/projects/infra-mirror-bucket).
- (TODO this is out of date) Currently only our Fastly configuration is feature-complete; if Fastly goes down then we failover to a CloudFront distribution that is only configured to serve a static mirror of GOV.UK.
  - Platform Reliability have begun work on a new CloudFront distribution that's able to serve content from origin by calling into a load balancer in front of our `cache` class nodes, but this new setup [doesn't have feature parity](https://docs.google.com/document/d/17_dfWvKNmqyLX1h_PPY6_Cd6IggrrSsP-Peh2De6JQk/edit) with our existing VCL-based Fastly service, and we have no plans to implement full feature parity with Fastly (as that's the purpose of this RFC).
  - This lack of feature parity presents us with several issues:
    - Lack of clarity or confidence around what works in Cloudfront (e.g. developers have to be aware of this when building things, SMT need to be reminded of impact failover will have, we may be more hesitant in triggering the failover, etc.)
    - Complexities also introduced downstream (e.g. if we failover and A/B testing stops working, data analysts now need to scrub out the failover period in their analysis)
    - We're less likely to drill the failover in Production to ensure it works (as it will cause a somewhat degraded experience)
    - It restricts our ability to consider a multi CDN strategy where traffic is split between two CDNs simultaneously 
- A number of the things that we currently handle at the CDN level might perhaps be better handled at other places in the stack, for example:
  - IP/JA3 allow/denylisting probably belongs in our WAF
  - A/B testing, feature flags, cache control and stripping cookies probably belong in the application layer

By moving things out of the CDN layer, we minimise the amount of duplicated effort in maintaining equivalent but different configurations for each CDN.

## Proposal

### Moving things from CDN to other places-->

Things that could be moved to WAF (TODO not sure if all of these actually _can_ be implemented there yet, need to do more research)

- JA3 denylisting: [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L178-L180) [2](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L195-L199)
- IP allowlisting on staging and production EKS [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L182-L187)
- Requiring HTTP Basic auth on integration [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L202-L207) [2](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L614-L620) (unless the user's IP is in the allowlist [3](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L154-L165))
- IP denylisting [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L189-L192)
  - n.b. This functionality is currently unused (the dictionary that the denylist is read from is empty)
- Silently ignore certain requests [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L223) [2](https://github.com/alphagov/govuk-cdn-config-secrets/blob/536de2171d17297c08a0a328df53a6b65002e2c4/fastly/fastly.yaml#L30-L39)
  - [Incident report](https://docs.google.com/document/d/12DzQsDeu7zUcICy9zVporjprX4qZFIrpOOWtYYRx-nk/edit)
- Serving an HTTP 404 response with a hardcoded template [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L579-L603) if the request URL matches `/autodiscover/autodiscover.xml` [2](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L226-L228)
  - Context: https://github.com/alphagov/govuk-cdn-config/pull/86
  - We can block this from our WAF, but that would change semantics (HTTP 403 vs 404), need to look into what impact that would have
  - Blocking at the WAF level would also remove the page template, but no users are likely to hit this URL anyway

Things that could be moved to Router(?):

- Redirecting `/security.txt` and `/.well-known/security.txt` to `https://vdp.cabinetoffice.gov.uk/.well-known/security.txt` [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L231-L233) [2](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L606-L612)
  - Could we set up an HTTP 301 redirect using Router? CDNs and browsers should cache 301s, so router should only be hit once

Things that could be moved to the application layer:

- Setting feature flags, such as showing recommended related links for Whitehall content [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L251)
  - On the old platform it was necessary to implement these flags in the CDN layer to prevent the need to deploy a new release every time we wanted to enable/disable a feature. A header is set on the backend request to indicate to the application whether the feature is enabled or disabled.
  - In the replatformed world, this becomes a lot easier: we can enable/disable features through environment variables (as opposed to headers), and then enabling or disabling becomes a matter of pushing a new commit to `govuk-helm-charts` and waiting a couple of minutes for Argo to pick it up.
  - Would need to [update the docs](https://docs.publishing.service.gov.uk/manual/related-links.html#suspending-all-suggested-related-links) as well
- A/B testing [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L524-L555) [2](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/_multivariate_tests.vcl.erb)
  - TODO: what are folks looking into in this space?

Things that need to remain in CDN:

- Require authentication for Fastly `PURGE` requests [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L171)
  - This doesn't need parity on Cloudfront
- Sorting query string params [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L236) and removing Google Analytics campaign params [2](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L239) to improve cache hit rate
- Stripping query string params only for the homepage and `/alerts` [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L256-L264)
  - TODO why do we do this?
- Automatic failover to static S3/GCS mirror if origin is unhealthy (only in staging and production) [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L266-L326)
  - This involves a lot of steps which are out of scope for this RFC, such as URL rewriting, customising cache behaviour, setting up grace periods and saint mode, and response code rewriting. If we move to Compute@Edge we will need to reimplement all of this (but it should be much easier to follow the logic then)
- Stripping the `Accept-Encoding` header if the content is already compressed [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L211-L215)
  - Context: https://github.com/alphagov/govuk-cdn-config/pull/379
  - n.b. [Fastly already normalises the `Accept-Encoding` header](https://developer.fastly.com/reference/http/http-headers/Accept-Encoding/#normalization), but it doesn't automatically remove it if the requested content is already compressed
- Controlling cache behaviour based on the `Cache-Control` header returned by origin [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L444-L455)
  - Manually set `Fastly-Cachetype` to `PRIVATE` if `Cache-Control: Private`: [1](https://github.com/alphagov/govuk-cdn-config/commit/03cb1fc5794658b89ed9f80ab5ca3c0b98a7afe7)
  - Explicit `pass` if `Cache-Control: max-age=0`: [2](https://github.com/alphagov/govuk-cdn-config/commit/54bf796f7c7543a893dbf14a8ca4fa1eae3253a1)
  - Explicitly `pass` if `Cache-Control: no-(store|cache)`: [3](https://github.com/alphagov/govuk-cdn-config/commit/fa56132e49d41595ba1681467adb828694cf0086)
  - It is unclear which (if any) of these remain necessary if we move to Compute@Edge (it's also unclear to me why Fastly doesn't respect them automatically 🤷‍♂️)
- Setting a request id header to allow requests to be traced through the stack [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L254)

Things that become unnecessary if we move to C@E:

- Explicitly marking HTTP 307 responses from origin as cacheable [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L459-L461)
  - Fastly VCL is built on an old version of Varnish which didn't do this by default
  - (TODO confirm if this is fixed in C@E)
- Enabling Brotli compression [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L329-L333) [2](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L388-L402)
  - TODO verify

Things that we could probably remove:

- Enforcing the use of TLS [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L218-L220) [2](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L561-L568)
  - Origin already performs an HTTP 301 redirect to the TLS version of the site. Browsers and CDNs should cache this response.

Undecided/TODO:

- Setting TLSversion request header for requests to Licensify for some reason [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L338-L340)
- GOV.UK accounts: Mapping from headers to cookies and back [1](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L504-L522) [2](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L350-L361) [3](https://github.com/alphagov/govuk-cdn-config/blob/55e587b238338caea1c7187c1f5d70cac8e5b104/vcl_templates/www.vcl.erb#L488-L492)
  - This is described in more detail in [RFC-134](https://github.com/alphagov/govuk-rfcs/blob/main/rfc-134-govuk-wide-session-cookie-and-login.md), and the discussion on the associated [PR](https://github.com/alphagov/govuk-rfcs/pull/134)
  - Code exists in our VCL to map between a cookie named `__Host-govuk_account_session` in user requests/responses, and the `GOVUK-Account-Session` and `GOVUK-Account-End-Session` headers in backend requests/responses, and to control the cache behaviour of these requests/responses
  - We have a couple of options here:
    - Pass the cookie through to origin and move the logic there (which goes against established precedent of stripping all cookies)
    - Reimplement this behaviour in both Compute@Edge and our Cloudfront distribution

<!--### Using Terraform to deploy both

TODO

### Retiring VCL

TODO-->

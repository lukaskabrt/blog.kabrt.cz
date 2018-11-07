Title: HTTPS and custom domain for a static website on Azure
Published: 11/01/2018
Tags: [Azure, Azure storage, Cloudflare, Static website]
---

In the [previous post](http://localhost:5080/posts/2018-10-hosting-static-website-on-azure), I set up my blog to run on Azure Static Website. However, it was accessible only on the domain provided by Azure `blogkabrtcz.z6.web.core.windows.net` and I'd like to use my own domain `blog.kabrt.cz`.

It is possible to setup Azure Static Website to use a custom domain, but it doesn't support SSL certificates for the custom domain (as of October 2018) ... and in these days SSL is must. You can use [Azure CDN](https://azure.microsoft.com/en-us/services/cdn) to overcome this limitation, but I decided to use another solution - [Cloudflare CDN](https://www.cloudflare.com/). They not only give you a free SSL certificate as Azure CDN but also you don't pay for the traffic between CDN and your visitors. You only pay for the traffic between Azure and Cloudflare edge nodes, which cache content of your website for the specified time. In my case, the savings are few cents per a month, but more visited websites can save a significant amount of money.

## Cloudflare CDN setup

When you create a new Cloudflare account, you get an empty dashboard. 

<figure class="figure w-100 text-center">
    <a href="/content/2018-10-custom-domain-https-for-static-website/01-dashboard.png" data-toggle="lightbox" data-gallery="2018-11-01-post">
        <img src="/content/2018-10-custom-domain-https-for-static-website/01-dashboard_t.png" class="figure-img img-fluid" alt="Cloudflare dashboard">
    </a>
</figure>

The first step is to add a domain to your account.

<figure class="figure w-100 text-center">
    <a href="/content/2018-10-custom-domain-https-for-static-website/02-add-domain.png" data-toggle="lightbox" data-gallery="2018-11-01-post">
        <img src="/content/2018-10-custom-domain-https-for-static-website/02-add-domain_t.png" class="figure-img img-fluid" alt="Cloudflare - Add domain">
    </a>
</figure>

You have to specify a second-level domain - in my case `kabrt.cz`. And select a plan for your site.

<figure class="figure w-100 text-center">
    <a href="/content/2018-10-custom-domain-https-for-static-website/03-plan.png" data-toggle="lightbox" data-gallery="2018-11-01-post">
        <img src="/content/2018-10-custom-domain-https-for-static-website/03-plan_t.png" class="figure-img img-fluid" alt="Cloudflare - Plans">
    </a>
</figure>

The free plan offers Cloudflare CDN and a free SSL certificate, so it is sufficient for this project. Higher plans offer prioritized support and advanced features, you might need in larger projects.

In order to take advantage of Cloudflare services, you need to use their DNS servers. Cloudflare tries to import up-to-date records from your current DNS servers, but you should double check, whether all records are loaded correctly.

<figure class="figure w-100 text-center">
    <a href="/content/2018-10-custom-domain-https-for-static-website/04-dns.png" data-toggle="lightbox" data-gallery="2018-11-01-post">
        <img src="/content/2018-10-custom-domain-https-for-static-website/04-dns_t.png" class="figure-img img-fluid" alt="Cloudflare - DNS">
    </a>
    <figcaption class="figure-caption text-center">Imported DNS records</figcaption>
</figure>

<figure class="figure w-100 text-center">
    <a href="/content/2018-10-custom-domain-https-for-static-website/05-dns-orig.png" data-toggle="lightbox" data-gallery="2018-11-01-post">
        <img src="/content/2018-10-custom-domain-https-for-static-website/05-dns-orig_t.png" class="figure-img img-fluid" alt="Original DNS records">
    </a>
    <figcaption class="figure-caption text-center">Original DNS records</figcaption>
</figure>

If any records are missing, you should add them manually, otherways some services on your domain might stop working.

The last step of site setup is the actual switching to Cloudflare DNS provider. It is done by changing nameservers to the ones provided by Cloudflare.

<figure class="figure w-100 text-center">
    <a href="/content/2018-10-custom-domain-https-for-static-website/06-ns.png" data-toggle="lightbox" data-gallery="2018-11-01-post">
        <img src="/content/2018-10-custom-domain-https-for-static-website/06-ns_t.png" class="figure-img img-fluid" alt="Cloudflare - NS setup">
    </a>
</figure>

If you are not sure, how to change nameservers for your domain, you should consult your domain registrar. It might take some time until settings propagate, but when the process is completed, Cloudflare changes the status of your site to 'Active' and you can start using their services.

## Azure Static Website - custom domain binding

You can setup a custom domain for your website in the Azure portal. Open the Azure storage account which contains your website and navigate to the 'Custom domain' tab.

<figure class="figure w-100 text-center">
    <a href="/content/2018-10-custom-domain-https-for-static-website/07-custom-domain.png" data-toggle="lightbox" data-gallery="2018-11-01-post">
        <img src="/content/2018-10-custom-domain-https-for-static-website/07-custom-domain_t.png" class="figure-img img-fluid" alt="Azure Static Website - custom domain binding">
    </a>
</figure>

Azure offers two methods to set up a custom domain. You can choose any of them depending on your needs. But there is a small caveat. By default, Cloudflare obfuscates a CNAME origin and returns the IP address of the referenced server in the A record, Azure doesn't like it and refuses to validate the custom domain.

Every DNS record on Cloudflare DNS can work in one of these modes:

* Records that display an orange cloud icon are accelerated and protected by Cloudflare
* Records that display a grey cloud icon will bypass Cloudflare, using only Cloudflare DNS
* Records that do not display a cloud icon must bypass Cloudflare, using only Cloudflare DNS

<p></p>

<figure class="figure w-100 text-center">
    <a href="/content/2018-10-custom-domain-https-for-static-website/cname-orange.png" data-toggle="lightbox" data-gallery="2018-11-01-post">
        <img src="/content/2018-10-custom-domain-https-for-static-website/cname-orange.png" class="figure-img img-fluid" alt="CNAME - protected">
    </a>
    <figcaption class="figure-caption text-center">Accelerated and protected by Cloudflare</figcaption>
</figure>

<figure class="figure w-100 text-center">
    <a href="/content/2018-10-custom-domain-https-for-static-website/cname-gray.png" data-toggle="lightbox" data-gallery="2018-11-01-post">
        <img src="/content/2018-10-custom-domain-https-for-static-website/cname-gray.png" class="figure-img img-fluid" alt="CNAME - only DNS">
    </a>
    <figcaption class="figure-caption text-center">Using only Cloudflare DNS</figcaption>
</figure>

If the record displays the orange cloud, the traffic goes through Cloudflare infrastructure and CNAME records are obfuscated. To add the custom domain on Azure, you have to change the mode to the 'gray cloud' for your website domain, validate the domain in the Azure Portal and then re-enable all Cloudflare features by switching back to the 'orange cloud' mode. You switch modes simply by clicking on the cloud icon. 

Once the domain is validated on Azure, you can access your site on the custom domain.

<figure class="figure w-100 text-center">
    <a href="/content/2018-10-custom-domain-https-for-static-website/08-working-domain.png" data-toggle="lightbox" data-gallery="2018-11-01-post">
        <img src="/content/2018-10-custom-domain-https-for-static-website/08-working-domain_t.png" class="figure-img img-fluid" alt="Working custom domain">
    </a>
</figure>

## SSL

Cloudflare automatically issues an SSL certificate for your domain and serves the web over HTTPS if the traffic goes through their servers. Various SSL settings can be tweaked on the 'Crypto' page of your Cloudflare dashboard.

<figure class="figure w-100 text-center">
    <a href="/content/2018-10-custom-domain-https-for-static-website/09-crypto.png" data-toggle="lightbox" data-gallery="2018-11-01-post">
        <img src="/content/2018-10-custom-domain-https-for-static-website/09-crypto_t.png" class="figure-img img-fluid" alt="Working custom domain">
    </a>
</figure>


> Be aware! If you set a DNS record to bypass Cloudflare, HTTPS will stop working

## Statistics

After several hours, Cloudflare will start displaying interesting statistics. You can check numbers of cached and uncached requests, amount of cached and uncached bandwidth and so on. 

<figure class="figure w-100 text-center">
    <a href="/content/2018-10-custom-domain-https-for-static-website/10-analytics.png" data-toggle="lightbox" data-gallery="2018-11-01-post">
        <img src="/content/2018-10-custom-domain-https-for-static-website/10-analytics_t.png" class="figure-img img-fluid" alt="Working custom domain">
    </a>
</figure>

And remember you don't pay for the cached traffic, because it is served from the Cloudflare CDN, not from your Azure Storage account.
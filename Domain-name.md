### Setting up a domain name

This can be done via AWS Route 53, but in this case [Google Domains](https://domains.google/#/) was used instead. Pretty straightforward. Look-up names, add to cart, and pay.

####Pointing Domain to AWS instance
As mentioned previously, it may be better to get a static Elastic IP from AWS and associate it with an instance. Regardless, assuming that there is an active instance running on AWS, go to EC2 > Instances, and select the desired instance. Look for the IPv4 public address and copy it. Note that Elastic IP does not seem to [support IPv6](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html).

After finding the instance public address, go to Google Domains and select the domain of interest from "My domains". Look for "DNS" on the left hand navigation panel. Scroll to the bottom and look for "Custom resource record".

The custom resource record takes several arguments: Name, type, time-to-live (TTL), and a data field.

The name can be used to specify sub-domains, but in this instance, we are only interested in the root domain, which is indicated by an "@" symbol, and the "www" sub-domain. An "A" record will be used for both the root and sub-domain, which maps a domain name to an IPv4 address. Lastly, in the data field, just input the public IPv4 address obtained from AWS earlier. Add the record and wait. After some time, the domain should map to the EC2 instance.

Alternatively, a CNAME, which maps an alias domain name to a true domain name, can also be considered for the "www" sub-domain, which is discussed  [here](https://serverfault.com/questions/223560/www-a-record-vs-cname-record).
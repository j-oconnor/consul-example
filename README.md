# Intro
Walkthrough of sample blue/green deploy process using consul service tags and KVP tag selector


## Setup

Ensure you have consul agent and consul-template installed.  I'm testing with consul v0.6.3 and consul-template v0.12.0.

Start consul in server mode advertising localhost
`consul agent -server -dev -advertise 127.0.0.1`

## Demo

We'll register some fake service endpoints to simplify the test with the v1 tag.

`while read i; do curl http://localhost:8500/v1/catalog/register -X PUT -d "${i}"; done < dummy-v1.json`  

Set the active tag version for "hello" app.

`curl "http://localhost:8500/v1/kv/hello/activeVersion?dc=dc1" -X PUT -d "v1"`

Check the output of consul query with the following:

`curl "http://localhost:8500/v1/health/service/hello?pretty&tag=v1"`

_Note:  For the purpose of this demo, we've not declared any healthchecks, so nodes are constantly "healthy"._

Check the output consul-template.

`consul-template -dry -once -template=template.ctmpl -consul=127.0.0.1:8500`

_Note:  This is not a complete/valid Nginx config, but represents the important bits_

Now we'll represent a deployment of a new revision by adding a new set of "hello" v2 nodes.

`while read i; do curl http://localhost:8500/v1/catalog/register -X PUT -d "${i}"; done < dummy-v2.json`

Consul now has both sets of services active/healthy.

`curl "http://localhost:8500/v1/health/service/hello?dc=dc1&pretty"`

*BUT* our consul template still only prints the v1 instances (since the hello/activeVersion key hasn't changed)

`consul-template -dry -once -template=template.ctmpl -consul=127.0.0.1:8500`

Now we update our activeVersion kv to "v2" to switch to the new deployment.

`curl "http://localhost:8500/v1/kv/hello/activeVersion?dc=dc1" -X PUT -d "v2"`

And our consul-template now shows upstreams set to V2 nodes.

`consul-template -dry -once -template=template.ctmpl -consul=127.0.0.1:8500`

If we don't like the deploy, we can still rollback to v1 at this point with a simple KV update.

`curl "http://localhost:8500/v1/kv/hello/activeVersion?dc=dc1" -X PUT -d "v1"`

# Prout

_"Has your pull request been deployed yet?"_

Tells you when your pull-requests are live. Tells you when they're not, and should be.

Prout comes from the philosophy that:

    Developers are responsible for checking their changes on Production

This becomes more important, and _easier_ once you move to a Continuous Deployment
release process. Important, because now a developer can break the site simply by
hitting 'Merge' on a pull request - but also easier because with such a small delay
(ie less than 10 minutes) between merging the work and having it ready to view in a
Production setting, the developer is in a much better place to review their work;
it's still fresh in their mind.

While everyone on your team may agree with this philosophy, that 10 minute lag
between merge and deploy can be enough time for a developer like me to get distracted
("Look, shiny thing!" or, more realistically, "What's the next bit of work?") and
forget about promptly reviewing their changes on Production.

Prout simply notifies developers in their pull request that the code has been _seen_
in Production (a slightly stronger statement than simply saying it's been deployed).


# Configuration

Follow the 4-step program:

1. Give [prout-bot](https://github.com/prout-bot) write-access to your repo (so it can set labels on your pull request)
2. Add one or more .prout.json config files to your project
3. Add callbacks to prout - ie a GitHub webhook and ideally also a post-deploy hook
4. Expose the commit id of your build on your deployed site

### Add config file

Add a `.prout.json` file to any folder you want monitored in your repo:

```
{
  "checkpoints": {
    "DEV": { "url": "http://dev.mysite.com/", "overdue": "10M" },
    "PROD": { "url": "http://mysite.com/", "overdue": "1H" }
  }
}
```

When a pull-request changes a file anywhere under that folder, Prout will scan the
checkpoints defined in your config file, and update the pull-request with labels
and a comment as appropriate. The url you specify in the checkpoint will be fetched,
and the contents of the response will be read- so long as you embed the commit id
that response, Prout will be able to work out whether or not the PR has been deployed.

### Add callbacks

Add Prout-hitting callbacks to GitHub and (optionally) post-deploy hooks to your deployment systems
so Prout can immediately check your site
[(full list of supported callbacks)](https://github.com/guardian/prout/blob/master/conf/routes).

##### GitHub

Add a [GitHub webhook](https://developer.github.com/webhooks/creating/#setting-up-a-webhook)
with these settings:

* Payload URL : `https://prout-bot.herokuapp.com/api/hooks/github`
* Content type : `application/json`

The hook should be set to activate on `Pull Request` events.

##### Post-deploy hooks

RiffRaff, Heroku etc, can all trigger an update by hitting `https://prout-bot.herokuapp.com/api/update/[owner]/[repo]`
(your repo lives on `https://github.com/[owner]/[repo]`) post-deploy.

### Expose the commit id

You must embed the commit id in your site - we do this on
[membership.theguardian.com](https://membership.theguardian.com/)
for instance.

I use the `sbt-buildinfo` plugin to store the Git commit id in my stored artifact, and then expose
that value on the production site. The ugly-looking SBT config is:

```
buildInfoKeys := Seq[BuildInfoKey](
      name,
      BuildInfoKey.constant("gitCommitId", Option(System.getenv("BUILD_VCS_NUMBER")) getOrElse(try {
        "git rev-parse HEAD".!!.trim
      } catch {
          case e: Exception => "unknown"
      }))
    )
```

# Run your own instance of Prout

[![Deploy](https://www.herokucdn.com/deploy/button.png)](https://heroku.com/deploy?template=https://github.com/guardian/prout)

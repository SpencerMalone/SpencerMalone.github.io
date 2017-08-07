---
title: Jenkins Shared Groovy Library Integration Testing
date: 2017-08-07 00:00:00 Z
categories:
- jenkins
layout: single
---

I've been playing with [JenkinsPipelineUnit](https://github.com/lesfurets/JenkinsPipelineUnit) for testing my shared groovy library in Jenkins, but found that it wasn't quite filling the niche that I wanted. Unit tests are good, but I needed to know if a new custom step would play nicely with our plugins, whether a new plugin would play nicely with our shared library, or whether a new entire jenkins version would play nicely with our shared library. With that in mind, I decided to look at how traditional plugins do integration testing.

I ended up slowly stumbling into [the unit testing wiki page for Jenkins](https://wiki.jenkins.io/display/JENKINS/Unit+Test), and realized that this test harness was probably the best way forward. I needed to be able to...

1. Specify my jenkins version, and all my plugin versions
2. Auto generate that list of jenkins versions and plugin versions
3. Run tests that load my shared library in it's current local state into some dynamically written jobs

So, I started with #1 on that list. I've fiddled about with using gradle to in my pipeline unit test project, so I decided to keep going down that path. Imported groovy, spock, jenkins test harness, and the jenkins version. Next, I knew that I needed a way to auto generate that list of plugins to use with Gradle. For now, I have a rather long groovy script that just generates the entire build.gradle file by pulling info from the Jenkins management groovy console. You can find that here: https://github.com/SpencerMalone/JenkinsPipelineIntegration/blob/master/generate-gradle-build.groovy. It has some odd logic, because we needed to pull both the .jar file for the plugins, and the .hpi to be loaded into our dynamic jenkins instance.

Next, I needed to load my library into the dynamic instance of jenkins. Plugins traditionally create their dynamic jenkins instance in tests by creating a "JenkinsRule". That was easy enough...

{% highlight groovy %}
@Rule JenkinsRule j = new JenkinsRule()
{% endhighlight %}

But now I needed to load the library in. Luckily, we have a large bank of tests to look at from the actual groovy shared library plugin found [here](https://github.com/jenkinsci/workflow-cps-global-lib-plugin. When looking at https://github.com/jenkinsci/workflow-cps-global-lib-plugin/blob/642d2b713cd9e30be05c03d5850324fa927cc152/src/test/java/org/jenkinsci/plugins/workflow/libs/ResourceStepTest.java), you can see that they're using a sample git repository, injecting in a file into the /vars folder (used for setting up custom steps), and then creating a Jenkins job that loads the library.

In our real Jenkins environment, we load our internal library implicitly from the master branch, so I wanted to replicate that functionality, but instead of using a sample git repository with files injected, I wanted to use the current working path. Luckily, git can just load folder references as well as URLs, so we'll rely on that to setup our library in the dynamic jenkins instance.

{% highlight groovy %}
    def lib = new LibraryConfiguration("rsg", new SCMSourceRetriever(new GitSCMSource(null, System.getProperty("user.dir"), "", "*", "0", true)))
{% endhighlight %}

Next, we need to set the library to implicitly load the current set of files. For this, I create a custom branch that will be relied on as being the current working state of our local copy of the repository, then set the custom library to implicitly load that branch.

{% highlight groovy %}
    def proc = Runtime.getRuntime().exec("git branch -m test-branch")
    proc.waitFor()
    lib.setImplicit(true)
    lib.setDefaultVersion('test-branch')
    GlobalLibraries.get().setLibraries(Collections.singletonList(lib))
{% endhighlight %}

Note that the custom branch has to be deleted between runs! Ex: `git branch -D test-branch || true`
I wrapped that up in a bash script that is used for running the tests, although it could be just as easily popped into your build.gradle.

In the end, my base test case class looks like this:

{% highlight groovy %}
package testSupport

import org.jvnet.hudson.test.JenkinsRule;
import hudson.model.*;
import org.junit.Rule

import org.jenkinsci.plugins.workflow.cps.CpsFlowDefinition;
import org.jenkinsci.plugins.workflow.job.WorkflowJob;
import spock.lang.Specification
import jenkins.plugins.git.GitSCMSource;
import org.jenkinsci.plugins.workflow.libs.*
/**
 * A base class for Spock testing using the pipeline helper
 */
class PipelineTestBase extends Specification {

    @Rule JenkinsRule j = new JenkinsRule()

    /**
     * Do the common setup
     */
    def setup() {

        def lib = new LibraryConfiguration("rsg", new SCMSourceRetriever(new GitSCMSource(null, System.getProperty("user.dir"), "", "*", "0", true)))
        def proc = Runtime.getRuntime().exec("git branch -m test-branch");
        proc.waitFor();
        lib.setImplicit(true)
        lib.setDefaultVersion('test-branch')
        GlobalLibraries.get().setLibraries(Collections.singletonList(lib));
    }
}
{% endhighlight %}

Which can be used like this:

{% highlight groovy %}
package tests.library

import testSupport.PipelineTestBase
import org.jenkinsci.plugins.workflow.cps.CpsFlowDefinition;
import org.jenkinsci.plugins.workflow.job.WorkflowJob;
import hudson.model.Result;


class isPRTestSpec extends PipelineTestBase {

    def "isPR returns true when env.BRANCH_NAME starts with PR-"() {

        given:
            WorkflowJob p = j.createProject(WorkflowJob.class, "p");

        when:
            p.setDefinition(new CpsFlowDefinition("""
                node(){ 
                    env.BRANCH_NAME = 'PR-123'; 
                    if(!isPR()) {
                        sh 'exit 1'
                    } 
                }"""
            ));

        then:
            def build = p.scheduleBuild2(0)
            def log = j.assertBuildStatusSuccess(build)
            j.assertLogContains("node",log) 
    }
}
{% endhighlight %}

If you want, the whole skeleton for this style of groovy shared library integration test can be found here: [https://github.com/SpencerMalone/JenkinsPipelineIntegration](https://github.com/SpencerMalone/JenkinsPipelineIntegration)

Major props to [https://github.com/macg33zr/pipelineUnit](https://github.com/macg33zr/pipelineUnit). I'd never really played with Spock before this, but so far have been loving it!

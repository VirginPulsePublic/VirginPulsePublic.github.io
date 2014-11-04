---
layout: post
title:  "Dependency Injection and Gamification - Part 1"
date:   2014-03-16 14:46:00 +0500
author: Steve Shi
comments: true
---

Dependency injection and inversion of control are a software design pattern that allows the removal of hard coded dependencies and makes it possible to change these dependencies at either run time or compile time.

I will be using the excellent stash-badgr to demonstrate how dependency injection works in modern software design.

Some context: Stashbadgr is a gamification component that awards Jira users with badges when they check code into Stash. Stashbadgr uses Spring’s dependency injection (DI) framework to inject at runtime various Achievement classes via configuration file.  However, the design pattern of DI is independent of any framework, so  bear with me even if you’ve never used Spring.

<!--more-->

First, understand where the dependencies are.  In Stashbadgr, the dependencies exist between Achievement Manager and Achievements.  The use case is to be able to process an event (when Changesets are committed ) with AchievementManager, which in turn calls the associated Achievement classes to check various business rules. These business rules are subject to change and new rules can be added in the future, and we don’t want to have to change our processor code in order to accommodate the new or revised rules.  This processing is inside the changeset processing class:  stash-badgr / src / main / java / nl / stefankohler / stash / badgr / idx / BadgrChangesetIndex.java:

{% highlight java %}
private void processObject(AchievementContext achievementContext, Changeset changeset,
                               Object validator, Achievement.AchievementType achievementType) {
    try {
        for (Achievement achievement : achievementManager.getAchievements(achievementType)) {
            if (achievement.isConditionMet(validator)) {
                achievementContext.incrementCount(achievement, changeset.getAuthor(), changeset);
            }
        }
    } catch (Exception e) {
        //error handling
    }
}
{% endhighlight %}

As you can see, it loops through all the Achievements associated with a given achievementType, (in our case, we will be using Action ids)  and these Achievements are managed via AchievementManager.

Next, let’s take a look at how Achievements get registered with AchievementManager dynamically at run time via dependency injection.

Look at stash-badgr / src / main / java / nl / stefankohler / stash / badgr / AchievementRegisterPostProcessor.java.  This is a class called after a bean is instantiated by Spring, because it implements BeanPostProcessor , and its constructor has a @Autowired annotation so Spring knows how to instantiate it.

Look at the postProcessAfterInitialization method, and you can see it’s adding the Achievement classes to the AchievementManager.

{% highlight java %}
 * The AchievementRegisterPostProcess is triggered by the auto-scanning
 * of  Achievement annotated classes. If the new bean is of the type
 * {@link Achievement} then it will be registered by the {@link AchievementManager}.
 * 
 * @author Stefan Kohler
 * @since 1.0
 */
@Component
public class AchievementRegisterPostProcessor implements BeanPostProcessor {

    private final AchievementManager achievementManager;

    /**
     * Constructs the PostProcessor.
     * @param achievementManager the {@link AchievementManager} to register the Achievements to.
     */
    @Autowired
    public AchievementRegisterPostProcessor(AchievementManager achievementManager) {
        this.achievementManager = achievementManager;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof Achievement) {
            achievementManager.register((Achievement) bean);
        }
        return bean;
    }
}
{% endhighlight %}

So, all that’s left (in simplistic terms) is to have the service automatically instantiate various Achievement beans at start-up time, then the framework will always register these classes to AchievementManager, and do so dynamically without hardcoding where Achievement classes are.   Let’s go take a look at the configurations:stash-badgr / src / main / resources / META-INF / spring / plugin-context.xml and see how that’s done.

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context-2.5.xsd">

    <context:component-scan base-package="nl.stefankohler.stash.badgr">
      <context:include-filter type="annotation" expression="nl.stefankohler.stash.badgr.Achievement"/>
    </context:component-scan>

</beans>
{% endhighlight %}

include-filer is a Spring component scan directive. The pattern above says to match anything that has an annotation type of nl.stefankohler.stash.badgr.Achievement And we can see that in the Achievement classes, they all have something like this:
@Achievement
public class BadgrBadgrBadgr extends AbstractAchievement

Finally, a summary on how everything is tied together:

1. !hen the service starts, the following classes are instantiated by the atlassian service, including DefaultAchievementManager (via a different configuration file in

stash-badgr / src / main / resources / atlassian-plugin.xml because stashbadgr is written as a jira plugin application)

2. Then, Spring scans the repository base nl.stefankohler.stash.badgr to instantiate additional classes that have autowired annotation; along with additional beans (the Achievements) that are specified in the configuration via a “filter”, and via a BeanPostProcessor the Achievement objects are registered with AchievementManager.

3. When Changesets are processed, the processor passes the changeset to the Achievement Manager

4. The Achievement Manager looks up all the registered Achievements by a specified event type, and these associated Achievements then reply whether or not the condition is met for awarding the achievement.  If condition is met, the Achievement is added to a collection for further processing.

5. Finally, when every associated Achievement is checked, the Achievements are grouped in an AchievementContext object, and changeset processor stash-badgr / src / main / java / nl / stefankohler / stash / badgr / idx / BadgrChangesetIndex.java calls updateAchievement to award the Achievements, which in turn, calls achievementManager.grantAchievement inside a transaction:

{% highlight java %}
public void onAfterIndexing(IndexingContext context) {
    AchievementContext achievementContext = context.get(BADGER_INDEXING_STATE);
    if (achievementContext != null) {
        achievementContext.updateAchievements();
    }
}
{% endhighlight %}

So as you can see, if you ever need to add a new Achievement, all you have to do is to create a new Achievement class, annotate it to be nl.stefankohler.stash.badgr.Achievement type, and associate the Achievement to an Action type. In stashbadgr, the related type is hardcoded inside the Achivement getType method as “CHANGESET”, but you can certainly see how this can be done via a database lookup, for example, using an event id for an Action, and associated that event id to a group of Achievements in the database.

Second, the Achievement business rules are encapsulated inside each Achievement isConditionMet method. This provides infinite design flexibility, since the implementation is “owned” by a class that can be injected as run time, we can easily create implementation inside the isConditionMet method in the future to look up business rules stored in the database associated with given Achievements.
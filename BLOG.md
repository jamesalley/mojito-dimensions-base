# Clean Your Plate: Avoiding Spaghetti Code in A/B/Multivariate Testing

Large applications. Numerous contributors. Continual A/B testing. Global development and deployment. All this leads to an engineering complexity problem. As applications become older, bigger, and go through more iterations, they tend to accumulate legacy code that eventually leads to paralysis. The initial owners have left, 7000 people have worked on it, and no one has its overall architecture in mind anymore.

Mojito-dimensions-base can't help you manage your global development team, should you have one, but it can make your testing regimens clean and sanitary.

Oftentimes A/B tests are created simply by forking code: if A, do this; if B, do that. If you run only a few tests at a time, and the tests are small, that may seem manageable. But over time, test forks tend to accumulate, because no one is willing to delete code for a test they are not familiar with. You end up with spaghetti code galore, with if/else statements all over the place. To make matters worse, code for any test can be spread across multiple files. Attempting to clean the system out of old, dead testing code can be a nightmare.

At Yahoo Search we wanted the ability to experiment ([bucket test](http://en.wikipedia.org/wiki/A/B_testing)) at all levels of the app, to be able to run any number of experiments concurrently, and not to disturb the mainline of our code, or worse, commit the sin of littering the mainline with spaghetti code.

Where then, should we write those experiments so that they can be combined dynamically, yet have no dependance on one another, be easily versionable, testable, maintainable, and reusable? The answer lies in how a mojito app is structured.

## The tree that masks the forest
Put roughly, a [mojito](http://developer.yahoo.com/cocktails/mojito/) app is a node package with a config file (in json or yaml) plus a set of directories, each corresponding to an independant widget - called "mojit". Each mojit has resources (files) for each: model, view, controller, client-side assets.

```yaml
search-apps-news
|-- mojits
|   |-- SearchBox/
|   |   |-- models/
|   |   |-- views/
|   |   |    `-- index.html
|   |   |-- controller.js
|   |   |-- assets/
|   |   ...
|   |-- SearchResult/
|   |   |-- models/
|   |   |-- views/
|   |   |-- controller.js
|   |   |-- assets/
|   |   ...
|   ...
|-- package.json
|-- ...
// the configuration file
`-- application.json
```

That's the base application, that's the forest. Now, say you want to try changing the view of the search box to add a button for some users to see how they react. You will want to change `search-apps-news/mojits/SearchBox/views/index.html`. Right? 

## Well no.
This is a recipe for a spaghetti code disaster when you have 40 experiments on that search box. It would get even worse if you needed to touch many files for your experiment. Besides, the logic that decides what user should get what view should be reusable and in the app framework (not your app). If you believe that, then [mojito-dimensions-base](https://github.com/yahoo/mojito-dimensions-base) just became your best friend. By including that package in the package containing your experiments, you can then create "mask packages" that mimic the structure of your app _only for those files that you want to override for that experiment_.

So to come back to experimenting on that extra button for some users, make a node package for your searchbox experiments that looks like this:

```yaml
mojito-dimensions-experiment_searchbox
 |-- extrabutton
 |   |-- mojits/
 |   |   `-- SearchBox/
 |   |      `-- views/
 |   |          `-- index.html
 |   `-- application.yaml
 ...
 |-- otherExperiment/
 ...
 |-- node_modules
 |   `-- mojito-dimensions-base/
 `-- package.json
```
Done. As you can see, the `extrabutton/` directory structure mirrors your base app, but only replaces one file: `mojits/SearchBox/views/index.html`.  An experiment can involve many file substitutions, but in this case, we only need to change a single file. `application.yaml` is the config that tells your app when to trigger that "mask" (that is, who are the "some users").

Et voila! If you now define `mojito-dimensions-experiment_searchbox` as a dependency of you app, the file loaded when your request matches the `extrabutton` configuration will be the one from the `extrabutton` package, _not_ the baseline!
You can now easily develop experiment packages that are completely separate from your baseline app. You can activate and deactivate at will, and merge easily when you determine you have a winner.

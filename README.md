# Screening Exercise: AutoML RLOS 2022

The readme describes the changes I did to complete the Screening exercise. And the repository contains the forked Vowpal Wabbit repository modified and the added snippets of code. The structure of the remainder of the README is as follows

- [Describing the Screening Exercise](#describing-the-screening-exercise)
- [My Approach to solve it](#my-approach)
    - [X] [Brute Force approach](#brute-force)
    - [X] [Efficient approach](#efficient-approach)

The following screening exercise is for the project [AutoML Extentions](https://vowpalwabbit.org/rlos/2022/projects#automl-extensions) where are working on integrating the ChaCha based online automl algorithm for obtaining optimal learning configuration.

## Describing the Screening Exercise

The screen exercise can be found [here](https://vowpalwabbit.org/rlos/2022/projects#screening-exercise). It states that we must report the iteration when the first champion switch occurs for a new seed number 11. Such that with this reported value set for `deterministic_champ_switch`, the unit test in `automl_first_champ_switch` passes with no error. Another factor is to explain the approach and process to achieve the solution.

## My Approach

After a deeper look into the cb_sim it was clear that we had to use the test_hooks with lambda functions to infer any information from the learner/simulator. And that these hooks run at the iteration step as specified in the callback map. So further I needed to look into the [reductions/automl.cc](vowpalwabbit/reductions/automl.cc) to understand the variables and functions in the [`interaction_config_manager`](vowpalwabbit/reductions/automl.cc#L154). It was very evident that the [`one_step`](vowpalwabbit/reductions/automl.cc#L122) function runs at every iteration and after the first batch of interactions we have a function named [`update_champ`](vowpalwabbit/reductions/automl.cc#L407) that checks the `better`/`worse` configs and checks if the champions should be swapped with the Challenger config or not. If in case the a champion is swapped with a challenger config we increment the variable [`total_champ_switches`](vowpalwabbit/reductions/automl.cc#L485). For our exercise we have to monitor the variable `total_champ_switches` and the iteration number when it first changes. As Automl Extention is implemented along side a `base_learner` so all the information is computed and stored in config_manager. And all the functions and values are isolated from the base_learner and the config_manager.

Hence to get the iteration number I proposed using either of the two method.
- [X] [Brute Force](#brute-force): Callback based check on every iteration of the simulator, for the variable `total_champ_switches` in the config_manager from automl.
- [X] [Efficient Approach](#efficient-approach): For the callback we get the config_manager point and check rather this check can be put inside the `one_step`((vowpalwabbit/reductions/automl.cc#L122) function only if in case the update champ function is called. This reduces the number of times the check is run and also prevents from requesting and making local copy of the config_manager.

#### Brute Force

For this approach we only require to define test_hook callback map with one lambda function that checks if the `total_champ_switches` is one for the first time.
```diff
diff --git a/test/unit_test/automl_test.cc b/test/unit_test/automl_test.cc
index 95a2c5413..87ee019be 100644
--- a/test/unit_test/automl_test.cc
+++ b/test/unit_test/automl_test.cc
@@ -82,6 +82,18 @@ BOOST_AUTO_TEST_CASE(automl_first_champ_switch)
   const size_t check_example = 200;
   callback_map test_hooks;
 
+  const size_t max_check = 250;
+  bool check_switch = false;
+  for(size_t inst=0; inst<max_check; inst++){
+    test_hooks.emplace(std::make_pair( inst, [inst, &check_switch](cb_sim&, VW::workspace& all, multi_ex&) {
+      VW::automl::automl<interaction_config_manager>* aml = aml_test::get_automl_data(all);
+      std::string msg = "First Champ switch Occured at: " + std::to_string((int) inst);
+      BOOST_CHECK_MESSAGE(aml->cm->total_champ_switches!=1 || check_switch, msg);
+      check_switch |= aml->cm->total_champ_switches==1;
+      return true;
+    }));
+  }
+  
   test_hooks.emplace(deterministic_champ_switch - 1, [&](cb_sim&, VW::workspace& all, multi_ex&)
```
#### Efficient Approach
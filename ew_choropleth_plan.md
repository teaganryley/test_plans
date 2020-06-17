# Test Plan

## Intro

This test plan was designed to be part of a CI/CD pipeline and therefore automated. I chose to write test cases in Gherkin because of its
clear grammar, its declarative style, and the potential for automation in Cucumber. In the spirit of the declarative style, I tried to leave 
implementation details out of the test plan. However, I do make recommendations about these tests might be implemented at the automation level.

## Smoke Tests

Smoke tests should be automated first. If these fail, the deployment fails.

```Gherkin
# smoke_test.feature
Feature: Smoke tests

  A user named Bea attempts to navigate to the data visualization page. 

  Scenario: Page loads
    Given Bea is on her home page
    When Bea navigates to the data visualization page
    Then the page loads    # Automation can simply check HTTP response status codes
    And the map appears    # Automation checks for the map SVG in the DOM
    And the map has data   # Automation checks if the external CSV was properly loaded

  Scenario: Important UI elements load
    Given Bea is on her home page
    When Bea navigates to the data visualization page
    Then the time series selection appears  # checks the DOM for UI elements
    * the animation speed selection appears
    * the pause button appears
    * the replay button appears
    * the district data field appears       # text box
    * the E&W data field appears
    * the map key appears

   # this will also indicate if the automation code needs to change due to UI refactor

```

## Functional Tests

### Happy Path Tests

High priority and should be automated. I wrote simple user stories to guide my test plan. 

```Gherkin
# choropleth.feature
Feature: Choropleth map colour change
  As a user named Bea, I want to see the map change colour every year when the animation plays.
  
  Scenario Outline:
    Given Bea has picked an animation speed 
    And Bea has chosen a time <time series>
    And Bea has selected a <district>
    When a <year> passes
    Then the district <colour value> changes

    Examples:
      | year | time series    | district |   colour value |
      | 1995 | current prices | Eden     |        #0000FF |    # hex value
      | 2015 | nominal        | Powys    | rgb(0, 0, 255) |    # rgb value

  # For happy path tests, I would only run tests on a single district per time series from 1995 to 2015
```
```Gherkin
# district_feedback.feature
Feature: District selection feedback

  As a user named Bea, I want to select a district and receive feedback from the data visualization.

  Background:
    Given Bea is on the page        # implies page state consisting of animation speed and time series 
    And the animation has started   # default values, or we can inject random selection into the automation

  Scenario: Map highlight
    When Bea selects a district on the map
    Then that district is highlighted on the map
    And district data is updated
    But animation does not restart

  Scenario: District data updates 
    When Bea selects a district on the map
    Then district data is updated
    # text box/time series permutations tested in priority 2

```
For the next feature, it was difficult to infer the functionality of the time series selection because the web page behaves 
inconsistently (see bugs). Generally, there are two questions to consider:
1. When the user selects a time series, should the map reset and the animation play from the beginning?
2. When the user selects a time series, should the district selection rest to no selection?

I decided that both should reset, based on the behaviour of the default selection. 

```Gherkin
# time_series.feature
Feature: Time series selection

  As a user named Bea, I want to change the time series and watch 
the new time series animation from the beginning. 

  Background: 
    Given Bea is on the page
    And animation has started

  Scenario: Map reset
    When Bea changes the time series
    Then the map resets
  
  Scenario: Animation restart
    When Bea changes the time series 
    Then the animation plays from the beginning

  Scenario: District unselected on the map
    Given Bea has selected a district on the map
    When Bea changes the time series
    Then the district is no longer highlighted

  Scenario: Text updates
    Given Bea has selected a district on the map
    When Bea changes the time series 
    Then the page title updates to time series
    And the district data fields update
    And the England and Wales fields update

```
The above assumptions hold for animation speed as well:
```Gherkin
# animation_speed.feature
Feature: Animation speed selection

  As a user named Bea, I want to change animation speed and watch the animation 
play from the beginning at the new speed. 

  Background: 
    Given Bea is on the page
    And animation has started

  Scenario: Speed increment behaviour
    When Bea increases speed
    Then the animation speed updates
    And the animation plays from the beginning
    And the map resets 

  Scenario: Speed decrement
    When Bea decreases speed
    Then the animation speed updates
    And the animation plays from the beginning
    And the map resets

  Scenario: User enters number
    When Bea enters an animation speed of her own
    Then the animation speed updates
    And the animation plays from the beginning
    And the map resets   

  Scenario: District reset
    Given Bea has selected a district on the map
    When Bea changes the animation speed
    Then the district is no longer highlighted
    And the district data fields reset to no selection

  # see priority 2 for boundary cases

```
```Gherkin
# pause_button.feature
Feature: Pause button

  As a user named Bea, I want to be able to pause and unpause the animation, without losing my progress.

  Scenario: Unpaused state -> paused state
    Given Bea is on the page
    And animation has started
    And Bea has selected a district on the map
    When Bea pauses the animation
    Then the map stops updating
    And the data fields stop updating
    But Bea's selections are not reset

  Scenario: Paused state -> unpaused state
    Given Bea is on the page
    And Bea has selected a district on the map    # order matters, in this case
    And the animation has been paused
    When Bea resumes the animation
    Then the map resume updating
    And the data fields resume updating
    But Bea's selections are not reset
    
```
```Gherkin
# replay_button.feature
Feature: Replay button

  As a user named Bea, I want to be able to replay the animation without losing my selections.

  Scenario: Bea hits replay mid-animation
    Given Bea is on the page
    And the animation has started
    And Bea has selected a district on the map
    When Bea hits the replay button
    Then the map resets
    And the animation plays from the beginning
    But Bea's selections are not reset
  
  Scenario: Bea hits replay at the end of the animation
    Given Bea is on the page
    And the animation has completed    # year = 2015, end of time series
    And Bea has selected a district on the map
    When Bea hits the replay button
    Then the map resets
    And the animation plays from the beginning
    But Bea's selections are not reset
  
```
### End-to-End Tests

Lower priority end-to-end tests that extend the coverage above. 
Since the data visualization consists of a single page and relatively few elements, the happy path tests cover most of the page's functionality.
However, these tests could bring value if the team has the need or bandwidth.

```Gherkin
# title_update.feature
Feature: Title update

  As a user named Bea, I want to see the title change when I select a new time series.

  Scenario Outline:
    Given Bea is on the page
    When Bea selects <times series>
    Then <title> changes

    Examples:
      | time series    |                                   title                                            |
      | Nominal prices | "England and Wales mean property prices by district, at nominal prices, 1995-2015" |
      | Current Prices | "England and Wales mean property prices by district, at current prices, 1995-2015" |

```

Depending on the cost associated with implementing the next two test, it might be better to let the Happy Path tests simply check
if the data updates, then let the CSV validation module handle CSV accuracy. Verifying the accuracy of on-page calculations would
live in the `EW_script.js` unit tests. Still, I have included these tests for posterity. 

```Gherkin
# district_accuracy.feature
Feature: District data accuracy

  As a user named Bea, I want to see my district data update accurately over time.

  Scenario Outline:
    Given Bea has picked an animation speed 
    And Bea has chosen a time <time series>
    And Bea has selected a <district>
    When a <year> passes
    Then the district <average price> changes
    And the district <annual increase> changes
    And the district <multiple of 1995 value> changes
    And the district <times mean FT earnings> changes

    Examples:
      | year | time series        | district | average price | annual increase | multiple of 1995 value | times mean FT earnings |
      | 1996 | current prices     |     Eden |        64,023 |              3% |                        |                        |
      | 2015 | compared with 1995 |     Eden |       210,133 |          -12.5% |                     2x |                        |

```

```Gherkin
# global_accuracy.feature
Feature: District data accuracy

  As a user named Bea, I want to see data update accurately over time for all of England and Wales.

  Scenario Outline:
    Given Bea has picked an animation speed 
    And Bea has chosen a time <time series>
    When a <year> passes
    Then the global <average price> changes
    And the global <annual increase> changes
    And the global <multiple of 1995 value> changes
    And the global <times mean FT earnings> changes

    Examples:
      | year | time series                  | average price | annual increase | multiple of 1995 value | times mean FT earnings |
      | 1995 | nominal prices               |        68,719 |                 |                        |                        |
      | 2015 | compared with local earnings |       291,826 |            5.1% |                        |                   8.9x |

```

```Gherkin
#animation_boundaries.feature
Feature: Animation spead boundaries

  As a user named Bea, I should not be able to select a speed greater than 20 000 ms, and less than 1 ms.

  Scenario: Over maximum value
    Given Bea is on the page
    When Bea changes the animation speed to 22000 ms
    Then an error is indicated    # drop down gains red outline, in this case
    And animation does not play

  Scenario: Under minimum value
    Given Bea is on the page
    When Bea changes the animation speed to 0
    Then an error is indicated
    And animation does not play
  
```

## Example Bug Reports

```
[Issue #001]- PapaParser resource blocked due to MIME type
[Link to User Story] [Bug]

Reproduction steps:
  1. Navigate to E&W animated choropleth
  2. Open console
  3. Observe error message: 
    "The resource [...] was blocked due to MIME type (text/plain) mismatch (X-Content-Type-Options: nosniff)."

Expected result: MIME type mismatch should be resolved
```

```
[Issue #002]- District data persists after changing time series
[Link to User Story] [Bug]

Reproduction steps:
  1. Navigate to E&W animated choropleth
  2. Select Northumberland district
  3. Select "Compared with 1995" time series
  4. Observe: In 1995, district data is reset
  5. Wait until 1996 arrives
  6. Observe: Northumberland appears in district data again

Expected result: When the user selects a new series, the district data should reset to 
                 no selection until the user makes a new selection.
```

```
[Issue #003]- District borders are not visible
[Link to User Story] [Feature request- Usability]

Summary: District borders are not sufficiently visible to the user when the map displays dark blue.

Reproduction steps:
  1. Navigate to E&W animated choropleth
  2. Select "compared with 1995" time series
  3. Select pause
  4. Observe: District borders blend into blue colour gradient

Expected result: User should be able to differentiate districts at all gradient colour values
```

```
[Issue #004]- Small districts cannot be selected due to map size
[Link to User Story] [Link to Issue #003] [Feature request- Usability]

Summary: Due to small map size, it is difficult for the user to differentiate between smaller districts.

Reproduction steps:
  1. Navigate to E&W animated choropleth
  2. Attempt to select specific districts surrounding London (eg// Newham) 
  3. Observe: Distinct districts are difficult to perceive or select

Expected result: Map size and scale should enable the user to differentiate between districts
```

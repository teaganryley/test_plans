# Test Plan

## Intro

This test plan was designed to be part of a CI/CD pipeline and therefore automated. I chose to write test cases in Gherkin because of its
clear grammar, its declarative style, and the potential for automation in Cucumber.

## Smoke Tests

Smoke tests should be automated first. If these fail, the deployment fails.

```Gherkin
# smoke_test.feature
Feature: Smoke tests

  A user named Bea attempts to navigate to the data visualization page. 

  Scenario: Page loading
    Given Bea is on her home page
    When Bea navigates to the data visualization page
    Then the page loads    # Automation can simply check HTTP response status codes
    And the map appears    # Automation checks external request to fetch CSV file

  #TODO: check that ui elements appear, or is that redundant? Wouldn't it be covered by the code review?

```

## User Acceptance Tests

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
    And the animation has started   # default time series and animation speed, or we can inject random selection into the automation

  Scenario: Map highlight
    When Bea selects a district on the map
    Then that district is highlighted on the map
    And district data is updated
    But animation does not reset

  Scenario: District data updates 
    When Bea selects a district on the map
    Then district data is updated
    # text box/time series permutations tested in priority 2
  
  Scenario: Animation does not reset
    When Bea selects a district on the map
    Then animation does not restart

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
    And the district data fields reset to no selection
    And the England and Wales fields updates

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

  Scenario: Paused state -> Unpaused state
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
### Additional End-to-End Tests

Since the data visualization consists of a single page and relatively few elements, the happy path tests cover most of the page's functionality.
However, here are some more extensive tests that could bring value.

```Gherkin
Feature: Title update

  As a user named Bea, I want to see the title change when I select a new time series
```
```Gherkin
Feature: District data

  As a user named Bea, I want to see my district data update over time

  Scenario:

year
  # TODO USE A TABLE?
```

pause text change
Favicon loads??
#boundary cases for animation speed
## Bug Reports
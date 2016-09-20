For improved code/functionality coverage UI end-2-end tests exist. Find more information below on how to set up Protractor and how to run the tests from your development environment.

# Set up Protractor

In a command line, navigate to the directory you want to install Protractor in.

Run the following commands:
1. npm install protractor (install Protractor via the Node Package Manager)
2. npm install selenium --save-dev (install Selenium driver via nmp)
3. npm install chromedriver --save-dev (install web driver for chrome via nmp)

Those should install without a problem.

4. Next, locate the Protractor executable (it should be under /node_modules/ in the directory you installed in)
5. protractor/bin/webdriver-manager update (this will update selenium webdriver)

This completes the installation. To make sure everything is installed properly start the local Selenium instance: protractor/bin/webdriver-manager start

You should see selenium start up in the terminal window.

# Running UI tests (WiP)

## From terminal
To run tests, you need to run the protractor executable and pass in the path to the protractor conf.js file which has your configuration

The example config file that comes with the installation should be somewhere around node_modules/protractor/example/conf.js

To run the tests, you'll use this command (although paths might be different depending on what folder you're in)
./bin/protractor <path-to-node_modules>/protractor/example/conf.js

## From IntelliJ (WiP)
Alternatively, you can run the tests from inside IntelliJ

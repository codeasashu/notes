## Build and Deployment Scripting

Script is required mostly to solve complex deployment problems such as
configuring your application, initializing data, configuring the infrastructure, operating systems, 
and middle- ware, setting up mock external systems, and so on

We can skip build tools for framework that does not emit any files except themselves.
That is, they can just be copy pasted to environments without any extra file/asset generation

### Principles of build and deployment scripting

1. #### Create a Script for Each Stage in Your Deployment Pipeline
    
    1. From the start, create a script which executes for all the deployment steps
    2. Commit script runs the tests, lints and code styles etc. 
    3. Acceptance test script runs acceptance tests
    4. Deployment script deploys the application to instances
    5. The steps which are not automated, can be just `echo` command (placeholders, to be implemented in future)
    6. The deployment script should cover the case of upgrading your application as well as installing it from scratch

2. #### Use the Same Scripts to Deploy to Every Environment
    1. it is essential to use the same process to deploy to every environment
    2. The differences b/w environment can be put into configuration management
    3. Leave out configuration values and retrieve those (from configuration management) during deployments
    4. Simplify the dependent applications to setup on local machines.
    5. Same process should be used by developers to setup deployment locally, as into production

3. #### Ensure the Deployment Process Is Idempotent
    1. deployment process should always leave the target environment in the same (correct) state,
        regardless of the state it finds it in when starting a deployment
    2. Start with environment your application needs to work with. Write down what is manually done (will be automated later)
    3. Deployment process then fetches the correct application version, and deploys
    4. Validate the configurations before making deployment. Check if DB, middleware, services are installed and running?
    5. Use Idempotent tools such as rsync, ansible, which puts the production in a desired state or rollsback to previous state

4. #### Testing Your Environment’s Configuration
    1. Environment should be validated separately before making deployment
    2. Test if DB are up and pinged
    3. Test if nginx can route traffic
    4. Test if supervisor is running
    5. Confirm that we can retrieve a record from our database

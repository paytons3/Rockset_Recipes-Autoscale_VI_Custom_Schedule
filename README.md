# Rockset Recipe: Auto-Scale Virtual Instance on a Custom Schedule

In this recipe, we'll be creating an "analytics" [Virtual Instance](https://docs.rockset.com/documentation/docs/virtual-instances) that will auto-scale up during peak workload hours and then auto-scale down after peak hours. We will implement this using [Scheduled Query Lambdas](https://docs.rockset.com/documentation/docs/query-lambdas#scheduled-query-lambdas).

Link to recipe on Rockset website: https://docs.rockset.com/documentation/recipes/auto-scale-vi-on-custom-schedule

**Requirements:**

- Python >= 3.6
- `pip install rockset time json`
- You must have [Multiple Virtual Instances](https://docs.rockset.com/documentation/docs/multiple-virtual-instances) for this recipe. Your existing VI must be a Dedicated VI size in order to create a secondary VI.

_The only things you have to change to successfully run this script are **rocksetApiKey**, **apiServerHost**,_ and _**webhookAuthorization**_.

## Step 1: Initialize the Client

Before we can start, we'll need to initialize the Rockset client. Create an API key in the [API Keys tab of the Rockset Console](https://console.rockset.com/apikeys). The region can be found in the dropdown menu at the top of the page.

```
# Set the API Keys as environmental variables in the shell via:
# export ROCKET_API_KEY='<insert key here>' and export OPENAI_API_KEY='<insert key here>'
rocksetApiKey = os.getenv("ROCKSET_API_KEY")
apiServerHost = Regions.usw2a1 #ex: rockset.Regions.usw2a1

rs = RocksetClient(host=apiServerHost, api_key=rocksetApiKey)
```

## Step 2: Define Parameters for Scheduled Query Lambdas

Here we will define the parameters that will be included in our [Scheduled Query Lambdas](https://docs.rockset.com/documentation/docs/query-lambdas#scheduled-query-lambdas). 

```
# Virtual Instance and Workspace parameters
virtualInstanceName = "testing"
virtualInstanceSize = "SMALL"
workspaceName = "commons"

# Parameter to create Webhook URLs for scaling VI up and down
urlPrefix = "https://api.usw2a1.rockset.com/v1/orgs/self/virtualinstances/"

# Parameters to create Scheduled Query Lambda for scaling VI up
scaleUpVIqueryLambdaName = "auto_scale_up_vi"
scaleUpDescription = "scale up `{virtualInstanceName}` VI"
cronScheduleScaleUp = "0 19 * * *" #UNIX cron format for every day at 19 UTC
scaleUpVISize = "MEDIUM"
scaleUpVIPayload = """
            {
            \"new_size\": \""""+scaleUpVISize+"""\"
            }
            """

# Parameters to create Scheduled Query Lambda for scaling VI down
scaleDownVIqueryLambdaName = "auto_scale_down_vi"
scaleDownDescription = "scale down `{virtualInstanceName}` VI"
cronScheduleScaleDown = "0 21 * * *" #UNIX cron format for every day at 21 UTC
scaleDownVISize = "SMALL"
scaleDownVIPayload = """
            {
            \"new_size\": \""""+scaleDownVISize+"""\"
            }
            """

webhookAuthorization = "apikey "+rocksetApiKey # 'apikey YOUR_ROCKSET_API_KEY'
```

## Step 3: Create Virtual Instance

Next we will create a [Virtual Instance](https://docs.rockset.com/documentation/docs/virtual-instances) (VI) named "analytics". This is the VI that we will scale up during peak workload hours and scale down at the end of peak hours.

```
def create_virtual_instance(rs, virtualInstanceName, virtualInstanceSize, mountType="STATIC"):
    try:
        print(f"Creating Virtual Instance `{virtualInstanceName}`...")
        api_response = rs.VirtualInstances.create(
            name=virtualInstanceName,
            mount_type=mountType,
            type=virtualInstanceSize,
        )
        print(api_response)
        print(f"Virtual Instance `{virtualInstanceName}` created!")
    except ApiException as e:
        print("Exception when creating Virtual Instance: %s\n" % json.loads(e.body))
```

```
create_virtual_instance(rs, virtualInstanceName, virtualInstanceSize)
```

## Step 4: Create API Endpoint URL

We now have to create the URL for the API endpoint that will be used to resize the VI. This has to be manually created since the URL must include the ID of the Virtual Instance.


```
def get_vi_id(rs, virtualInstanceName):
    try:
        api_response = rs.VirtualInstances.list(
        )
        vi_list = api_response.data
        for vi in vi_list:
            if vi.get("name") == virtualInstanceName:
                return vi.get("id")
    except ApiException as e:
        print("Exception when getting id for Virtual Instance: %s\n" % json.loads(e.body))
        return None
```

```
virtualInstanceID = get_vi_id(rs, virtualInstanceName)
scaleVIurl = create_webhookURL(urlPrefix, virtualInstanceID)
```

## Step 5: Create Query Lambdas

[Query Lambdas](https://docs.rockset.com/documentation/docs/query-lambdas) (QLs) are parameterized SQL queries that can be executed from a dedicated REST endpoint. In this step, we'll save an arbitrary SQL query of `SELECT 1` as the Query Lambdas for suspending and resuming the Virtual Instance. You must use a query that always returns _at least one document_ in the results of the executed QL. This is required in order to send a request to the webhook when running the Scheduled Query Lambdas we will create in the next step. The webhook request is what will be used to resize the VI.

```
def create_query_lambda(rs, workspaceName, queryLambdaName, description):
    description=description
    query = f"""
    SELECT 1
    """

    # Create QL
    try:
        print(f"Creating query lambda `{queryLambdaName}`...")
        api_response = rs.QueryLambdas.create_query_lambda(
            name=queryLambdaName,
            workspace=workspaceName,
            sql=QueryLambdaSql(
                query=query,
            ),
        )
        print(f"Query lambda `{queryLambdaName}` created!")
    except ApiException as e:
        print(f"Exception when creating query lambda: %s\n" % json.loads(e.body))
```

```
create_query_lambda(rs, workspaceName, scaleUpVIqueryLambdaName, scaleUpDescription)
create_query_lambda(rs, workspaceName, scaleDownVIqueryLambdaName, scaleDownDescription)
```

## Step 6: Create Scheduled Query Lambdas

You can schedule your QLs for automatic execution and configure certain actions to be taken based on the results of the executed query lambda using [Scheduled Query Lambdas](https://docs.rockset.com/documentation/docs/query-lambdas#scheduled-query-lambdas). We will now create two Scheduled QLs: one to scale up the VI and one to scale down the VI.

```
def created_scheduled_ql(rs, workspaceName, queryLambdaName, cronSchedule, webhookURL, webhookAuthorization, webhookPayload, numberOfExecutions=1, tagName='latest'):

    try:
        print(f"Creating scheduled query lambda for `{queryLambdaName}`...")
        api_response=rs.ScheduledLambdas.create(
        apikey=rocksetApiKey,
        cron_string=cronSchedule,
        ql_name=queryLambdaName,
        tag=tagName,
        total_times_to_execute=numberOfExecutions, # Comment out for unlimited number of executions.
        webhook_auth_header=webhookAuthorization,
        webhook_payload=webhookPayload,
        webhook_url=webhookURL,
        workspace=workspaceName
        )
        print(f"Scheduled query lambda for `{queryLambdaName}` created!")
        print(api_response)
    except ApiException as e:
        print(f"Exception when creating scheduled query lambda: %s\n" % json.loads(e.body))
```

```
created_scheduled_ql(rs, workspaceName, scaleUpVIqueryLambdaName, cronScheduleScaleUp, scaleVIurl, webhookAuthorization, scaleUpVIPayload)
created_scheduled_ql(rs, workspaceName, scaleDownVIqueryLambdaName, cronScheduleScaleDown, scaleVIurl, webhookAuthorization, scaleDownVIPayload)
```

## Step 7: Clean Up

You have now successfully set up a custom schedule to auto-scale your VI! As a courtesy to you, we've included a function to delete the Scheduled Query Lambdas, Query Lambda, and Virtual Instance that were created in this recipe at your leisure.

```
def clean_up_demo(rs, virtualInstanceName, virtualInstanceId, queryLambdaNames, scheduledQLrrns, max_attempts=10):

    # Deleting Schedules
    for scheduleRRN in scheduledQLrrns:
        try:
            print(f"Deleting schedule `{scheduleRRN}`...")
            api_response = rs.ScheduledLambdas.delete(
    				scheduled_lambda_id=scheduleRRN,
			)
        except ApiException as e:
            print(f"Exception when deleting schedule: %s\n" % json.loads(e.body))

        # Checking if Schedule is deleted
        for attempt in range(max_attempts):
            try:
                api_response = rs.ScheduledLambdas.get(
                    scheduled_lambda_id=scheduleRRN
                )
            except ApiException as e:
                if json.loads(e.body).get("message_key") == 'SCHEDULED_LAMBDA_NOT_FOUND': 
                    print(f'Schedule `{scheduleRRN}` deleted.')
                    break
            time.sleep(30)
  
    # Deleting Query Lambdas
    for queryLambda in queryLambdaNames:
        try:
            print(f"Deleting Query Lambda `{queryLambda}`...")
            api_response = rs.QueryLambdas.delete_query_lambda(
                query_lambda=queryLambda, 
                workspace=workspaceName
            )
        except ApiException as e:
            print(f"Exception when deleting Query Lambda: %s\n" % json.loads(e.body))

        # Checking if Query Lambda ia deleted
        for attempt in range(max_attempts):
            try:
                api_response = rs.QueryLambdas.get_query_lambda_tag_version(
                    query_lambda=queryLambda, 
                    workspace=workspaceName, 
                    tag="latest"
                )
            except ApiException as e:
                if json.loads(e.body).get("message_key") == "QUERY_NOT_FOUND": 
                    print(f"Query Lambda `{queryLambda}` deleted.")
                    break
            time.sleep(30)
    
    # Delete Virtual Instance
    try:
        print(f"Deleting Virtual Instance `{virtualInstanceName}`...")
        api_response = rs.VirtualInstances.delete(
            virtual_instance_id=virtualInstanceId
        )
        # Checking if Virtual Instance is deleted
        for attempt in range(max_attempts):
            try:
                api_response = rs.VirtualInstances.get(
                    virtual_instance_id=virtualInstanceId
                )
            except ApiException as e:
                if json.loads(e.body).get("message_key") == "VIRTUAL_INSTANCE_NOT_FOUND": 
                    print(f"Virtual Instance `{virtualInstanceName}` deleted.")
                    break
            time.sleep(30)
    except ApiException as e:
        print(f"Exception when deleting Virtual Instance: %s\n" % json.loads(e.body))
```

```
virtualInstanceID = get_vi_id(rs, virtualInstanceName)
scaleUpQLrrn = get_scheduled_ql_rrn(rs, scaleUpVIqueryLambdaName)
scaleDownQLrrn = get_scheduled_ql_rrn(rs, scaleDownVIqueryLambdaName)
clean_up_demo(rs, virtualInstanceName, virtualInstanceID, [scaleUpVIqueryLambdaName, scaleDownVIqueryLambdaName], [scaleUpQLrrn, scaleDownQLrrn])
```

## What's Next?

Want to learn more about what you can automate using Scheduled Query Lambdas? Check out the following:
- Blog: [5 Tasks You Can Automate in Rockset Using Scheduled Query Lambdas](https://rockset.com/blog/5-tasks-you-can-automate-in-rockset-using-scheduled-query-lambdas/)
- Workshop: [Automate Your Workflow with Rockset’s Scheduled Query Lambdas](https://www.youtube.com/watch?v=CuQYCE0Bbsc)
- Recipe: [Auto-Resume/Suspend VI on Custom Schedule](https://docs.rockset.com/documentation/recipes/auto-resumesuspend-vi-on-custom-schedule)
- Recipe: [Send Automated Email Reports](https://docs.rockset.com/documentation/recipes/send-automated-email-reports)

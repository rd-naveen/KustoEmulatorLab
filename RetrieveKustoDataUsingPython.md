### Steps to  query kusto clusters using python.

```
from azure.kusto.data import KustoClient, KustoConnectionStringBuilder
from azure.kusto.data.helpers import dataframe_from_result_table

KUSTO_URI = "https://kvc-m23a83j31rynmvx6wh.northeurope.kusto.windows.net/"

KCSB_DATA = KustoConnectionStringBuilder.with_interactive_login(
    KUSTO_URI)
	
KUSTO_CLIENT = KustoClient(KCSB_DATA)

QUERY = "StormEvents | count"

RESPONSE = KUSTO_CLIENT.execute_query(KUSTO_DATABASE, QUERY) // during this step interactive authentication will happen over the webbrowser.

df_ = dataframe_from_result_table(RESPONSE.primary_results[0])

print(df_)
```

 
<img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/7d8db8e1-1aa6-4685-a0c2-267edbbbf149" />

---------------------------------------------------------------------------
ReferenceError                            Traceback (most recent call last)
Cell In[1], line 5
      2 import pandas as pd
      3 import re
----> 5 df = Alteryx.read("#1")
      7 records = []
      9 # Very light filtering only

File c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages\ayx\export.py:35, in read(incoming_connection_name, debug, **kwargs)
     31 def read(incoming_connection_name, debug=False, **kwargs):
     32     """
     33     When running the workflow in Alteryx, this function will convert incoming data streams to pandas dataframes when executing the code written in the Python tool. When called from the Jupyter notebook interactively, it will read in a copy of the incoming data that was cached on the previous run of the Alteryx workflow.
     34     """
---> 35     return __CachedData__(debug=debug).read(incoming_connection_name, **kwargs)

File c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages\ayx\CachedData.py:295, in CachedData.read(self, incoming_connection_name)
    288     print(
    289         'Attempting to read in cached data for incoming connection "{}"'.format(
    290             incoming_connection_name
    291         )
    292     )
    294 # get the filepath of the data
--> 295 input_data_metadata = self.__getIncomingConnectionMetadata(
    296     incoming_connection_name
    297 )
    298 input_data_filename = input_data_metadata["filename"]
    299 # create datafile object
    300 # (by not specifying the fileformat paramter, it will assume the file
    301 # type from the file's extension)

File c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages\ayx\CachedData.py:76, in CachedData.__getIncomingConnectionMetadata(self, incoming_connection_name)
     74 # error if connection name is not a named key in the config json (dict)
     75 elif incoming_connection_name not in input_file_map:
---> 76     raise ReferenceError(
     77         "".join(
     78             [
     79                 'The input connection "{}" has not been cached'.format(
     80                     incoming_connection_name
     81                 ),
     82                 " -- re-run workflow to refresh the cached data.",
     83             ]
     84         )
     85     )
     86 else:
     87     return input_file_map[incoming_connection_name]

ReferenceError: The input connection "#1" has not been cached -- re-run workflow to refresh the cached data.

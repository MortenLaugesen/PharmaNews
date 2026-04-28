----------------------
TypeError                                 Traceback (most recent call last)
Cell In[5], line 49
     38         records.append({
     39             "message_id": message_id,
     40             "email_source_type": source,
   (...)
     44             "article_url_extraction_method": extraction_method
     45         })
     47 out_df = pd.DataFrame(records)
---> 49 Alteryx.write(out_df, 1)

File c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages\ayx\export.py:87, in write(pandas_df, outgoing_connection_number, columns, debug, **kwargs)
     83 def write(pandas_df, outgoing_connection_number, columns=None, debug=False, **kwargs):
     84     """
     85     When running the workflow in Alteryx, this function will convert a pandas data frame to an Alteryx data stream and pass it out through one of the tool's five output anchors. When called from the Jupyter notebook interactively, it will display a preview of the pandas dataframe. An optional 'columns' argument allows column metadata to specify the field type, length, and name of columns in the output data stream.
     86     """
---> 87     return __CachedData__(debug=debug).write(
     88         pandas_df, outgoing_connection_number, columns=columns, **kwargs
     89     )

File c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages\ayx\CachedData.py:641, in CachedData.write(self, pandas_df, outgoing_connection_number, columns, output_filepath)
    636 msg_action = "writing outgoing connection data {}".format(
    637     outgoing_connection_number
    638 )
    639 try:
    640     # get the data from the sql db (if only one table exists, no need to specify the table name)
--> 641     data = db.writeData(pandas_df_out, metadata=write_metadata)
    642     # print success message
    643     if outgoing_connection_number is not None:

File c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages\ayx\Datafiles.py:733, in Datafile.writeData(self, pandas_df, metadata)
    722     error_msg = msg_prefix.format(
    723         " ".join(
    724             [
   (...)
    730         )
    731     )
    732     print(error_msg)
--> 733     raise TypeError(error_msg)
    734 elif len(metadata) != len(pandas_df.columns):
    735     error_msg = msg_prefix.format(
    736         "metadata must have same number of columns as pandas_df"
    737     )

TypeError: [Datafile.writeData]: metadata arg is required for yxdb and expected to be dict like {'Field1': {'type': 'FixedDecimal', 'length': (8, 3), 'source': 'PythonTool:', 'description': 'my description'}, 'Field2': {...}}

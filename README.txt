No error is displayed on Batch Job Executions screen when Batch Runtime database is down


On the Batch Job Executions screen in the Admin Console, no error message is displayed when the database, used by the Batch Runtime Configuration, isn't running, or is otherwise inaccessible. All you see is an empty Batch Jobs list ("No items found"), just as you get when the database is running and there are no batch job execution results in the database.

This bug exists in Glassfish 4.1 and above.


Steps to reproduce the bug:

1) Start the server:

    asadmin start-domain
	
2) In the Admin Console, navigate to server->Batch->Configuration and set the "Data Source Lookup Name" to a DataSource for the database to be used for storing Batch Runtime job execution information. Do NOT use "jdbc/__TimerPool", as this uses Embedded JavaDB, which is not independently stoppable to the Glassfish server. Stop and start the server to make the configuration change effective.

3) Ensure that the database is running.

4) Deploy and then run (several times) a sample Batch application.

5) Go to server->Batch->Executions, and verify that there are batch job execution results, from running the batch application.

6) Stop the database.

7) Click somewhere else in the Admin Console, then go back to server->Batch->Executions. No execution results are displayed, but no error message is displayed to indicate a problem with connectivity to the database.

8) Re-start the server:

    asadmin restart-domain

9) In the Admin Console, go to server->Batch->Executions. Again, no execution results are displayed, but no error message is displayed to indicate a problem with connectivity to the database.

	  	  
Resolution:

In the "execute" method of the command class (org.glassfish.batch.AbstractListCommandProxy) used for listing batch job executions, there are some code paths which do not set the ActionExitCode in the ActionReport returned in the AdminCommandContext. In these cases, the exit code remains as the default value, ActionReport.ExitCode.SUCCESS, which results in failures being undetected - and so not reported on the Batch Job Executions screen.

I have created a Glassfish 4.1.1 patch below to correct these problems:


--- AbstractListCommandProxy.java	(revision 64112)
+++ AbstractListCommandProxy.java	(working copy)
@@ -107,5 +107,6 @@
         ActionReport subReport = null;
         if (! preInvoke(context, actionReport)) {
             commandsExitCode = ActionReport.ExitCode.FAILURE;
+            actionReport.setActionExitCode(commandsExitCode);
             return;
         }
@@ -133,9 +134,9 @@
                     postInvoke(context, subReport.getSubActionsReport().get(0));
                 } else {
                     actionReport.setMessage(subReport.getMessage());
-                    commandsExitCode = subReport.getActionExitCode();
                 }
             }
+            commandsExitCode = subReport.getActionExitCode();
         }
         actionReport.setActionExitCode(commandsExitCode);
     }




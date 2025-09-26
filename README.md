[2025-09-26 09:13:15] local.INFO: Job started for project: 342538849279  
[2025-09-26 09:13:15] local.INFO: Refresh_token In the Job >>>bmPPH3WZuiWmXs1BAN8plyWgtiwb0hcJFVarTC61TtAUZiL7OVFvYB3omvmZTBHb  
[2025-09-26 09:15:47] local.INFO: Job started for project: 342538849279  
[2025-09-26 09:15:47] local.INFO: Refresh_token In the Job >>>bmPPH3WZuiWmXs1BAN8plyWgtiwb0hcJFVarTC61TtAUZiL7OVFvYB3omvmZTBHb  
[2025-09-26 09:18:22] local.ERROR: Job failed for project 342538849279: App\Jobs\SyncBoxProject has been attempted too many times or run too long. The job may have previously timed out.  
[2025-09-26 09:18:23] local.ERROR: App\Jobs\SyncBoxProject has been attempted too many times or run too long. The job may have previously timed out. {"exception":"[object] (Illuminate\\Queue\\MaxAttemptsExceededException(code: 0): App\\Jobs\\SyncBoxProject has been attempted too many times or run too long. The job may have previously timed out. at C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Queue\\Worker.php:750)
[stacktrace]
#0 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Queue\\Worker.php(504): Illuminate\\Queue\\Worker->maxAttemptsExceededException()
#1 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Queue\\Worker.php(418): Illuminate\\Queue\\Worker->markJobAsFailedIfAlreadyExceedsMaxAttempts()
#2 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Queue\\Worker.php(378): Illuminate\\Queue\\Worker->process()
#3 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Queue\\Worker.php(172): Illuminate\\Queue\\Worker->runJob()
#4 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Queue\\Console\\WorkCommand.php(117): Illuminate\\Queue\\Worker->daemon()
#5 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Queue\\Console\\WorkCommand.php(101): Illuminate\\Queue\\Console\\WorkCommand->runWorker()
#6 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Container\\BoundMethod.php(36): Illuminate\\Queue\\Console\\WorkCommand->handle()
#7 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Container\\Util.php(40): Illuminate\\Container\\BoundMethod::Illuminate\\Container\\{closure}()
#8 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Container\\BoundMethod.php(93): Illuminate\\Container\\Util::unwrapIfClosure()
#9 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Container\\BoundMethod.php(37): Illuminate\\Container\\BoundMethod::callBoundMethod()
#10 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Container\\Container.php(653): Illuminate\\Container\\BoundMethod::call()
#11 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Console\\Command.php(136): Illuminate\\Container\\Container->call()
#12 C:\\home\\site\\wwwroot\\ccc\\vendor\\symfony\\console\\Command\\Command.php(298): Illuminate\\Console\\Command->execute()
#13 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Console\\Command.php(121): Symfony\\Component\\Console\\Command\\Command->run()
#14 C:\\home\\site\\wwwroot\\ccc\\vendor\\symfony\\console\\Application.php(1040): Illuminate\\Console\\Command->run()
#15 C:\\home\\site\\wwwroot\\ccc\\vendor\\symfony\\console\\Application.php(301): Symfony\\Component\\Console\\Application->doRunCommand()
#16 C:\\home\\site\\wwwroot\\ccc\\vendor\\symfony\\console\\Application.php(171): Symfony\\Component\\Console\\Application->doRun()
#17 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Console\\Application.php(94): Symfony\\Component\\Console\\Application->run()
#18 C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Foundation\\Console\\Kernel.php(129): Illuminate\\Console\\Application->run()
#19 C:\\home\\site\\wwwroot\\ccc\\artisan(37): Illuminate\\Foundation\\Console\\Kernel->handle()
#20 {main}
"} 

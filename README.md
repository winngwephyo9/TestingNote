In the log file of webjob on stagging server
[09/29/2025 01:09:59 > 29f187: SYS INFO] Status changed to Running
[09/29/2025 01:10:04 > 29f187: INFO] [2025-09-29 10:10:04][44] Processing: App\Jobs\SyncBoxProject
[09/29/2025 01:11:33 > 29f187: ERR ] PHP Fatal error:  Allowed memory size of 134217728 bytes exhausted (tried to allocate 20480 bytes) in C:\home\site\wwwroot\ccc\vendor\laravel\framework\src\Illuminate\Database\Connection.php on line 375
[09/29/2025 01:11:33 > 29f187: ERR ] PHP Fatal error:  Allowed memory size of 134217728 bytes exhausted (tried to allocate 4096 bytes) in C:\home\site\wwwroot\ccc\vendor\symfony\error-handler\Error\FatalError.php on line 85
[09/29/2025 01:11:33 > 29f187: SYS ERR ] Job failed due to exit code 255
[09/29/2025 01:11:33 > 29f187: SYS INFO] Process went down, waiting for 60 seconds
[09/29/2025 01:11:33 > 29f187: SYS INFO] Status changed to PendingRestart
[09/29/2025 01:12:33 > 29f187: SYS INFO] WebJob singleton lock is released
[09/29/2025 01:12:33 > 29f187: SYS INFO] WebJob singleton lock is acquired
[09/29/2025 01:12:33 > 29f187: SYS INFO] Run script 'queue-worker.cmd' with script host - 'WindowsScriptHost'
[09/29/2025 01:12:33 > 29f187: SYS INFO] Status changed to Running
[09/29/2025 01:12:36 > 29f187: INFO] [2025-09-29 10:12:36][44] Processing: App\Jobs\SyncBoxProject
[09/29/2025 01:12:37 > 29f187: INFO] [2025-09-29 10:12:37][44] Failed:     App\Jobs\SyncBoxProject



In the log file
[2025-09-29 10:05:54] local.INFO: box_login_time In session : 1759107951  
[2025-09-29 10:05:54] local.INFO: refresh_token in Session : MaNwtLhyVFkdqGrOkKfIZaKMUBEvdhrrZrDAQbtTJHpDYIn5eyL15gkGFUMvtdso  
[2025-09-29 10:05:54] local.INFO: url_path in Session : https://ohb-spoke-ccc-deployment.azurewebsites.net/ccc  
[2025-09-29 10:05:57] local.INFO: Job started for project: 342538849279  
[2025-09-29 10:05:57] local.INFO: Refresh_token In the Job >>>MaNwtLhyVFkdqGrOkKfIZaKMUBEvdhrrZrDAQbtTJHpDYIn5eyL15gkGFUMvtdso  
[2025-09-29 10:07:31] local.INFO: Job started for project: 342538849279  
[2025-09-29 10:07:31] local.INFO: Refresh_token In the Job >>>MaNwtLhyVFkdqGrOkKfIZaKMUBEvdhrrZrDAQbtTJHpDYIn5eyL15gkGFUMvtdso  
[2025-09-29 10:10:04] local.INFO: Job started for project: 342538849279  
[2025-09-29 10:10:04] local.INFO: Refresh_token In the Job >>>MaNwtLhyVFkdqGrOkKfIZaKMUBEvdhrrZrDAQbtTJHpDYIn5eyL15gkGFUMvtdso  
[2025-09-29 10:12:37] local.ERROR: Job failed for project 342538849279: App\Jobs\SyncBoxProject has been attempted too many times or run too long. The job may have previously timed out.  
[2025-09-29 10:12:37] local.ERROR: App\Jobs\SyncBoxProject has been attempted too many times or run too long. The job may have previously timed out. {"exception":"[object] (Illuminate\\Queue\\MaxAttemptsExceededException(code: 0): App\\Jobs\\SyncBoxProject has been attempted too many times or run too long. The job may have previously timed out. at C:\\home\\site\\wwwroot\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Queue\\Worker.php:750)
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

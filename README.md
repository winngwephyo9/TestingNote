/**
     * 【重要】ジョブがタイムアウトするまでの秒数。
     * 0は無制限を意味します。同期は時間がかかるため、無制限に設定します。
     * @var int
     */
    public $timeout = 0;

    /**
     * 【重要】ジョブが失敗した場合の最大試行回数。
     * @var int
     */
    public $tries = 1;

    /**
     * 【重要】タイムアウトしたジョブを自動的に失敗としてマークするかどうか。
     * @var bool
     */
    public $failOnTimeout = true;

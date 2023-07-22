---
title: "Laravel Tenancy Testing"
date: 2023-07-22T20:02:10+08:00
draft: true
tags: ["laravel", "tenancy", "laravel tenancy"]
---

近期公司專案需要使用到 [multi-tenancy](https://zh.wikipedia.org/zh-tw/%E5%A4%9A%E7%A7%9F%E6%88%B6%E6%8A%80%E8%A1%93) 架構，所以我想分享一下使用這個架構時的測試經驗。

我使用了 [Laravel Tenancy V3](https://github.com/archtechx/tenancy) 套件來實現多租戶功能。在寫測試時，我們依照這個套件並新增測試。

## Testing
新增 Trait 方便測試與使用

```php
<?php

namespace Tests\Traits;

use App\Models\TenantModel;
use Illuminate\Contracts\Console\Kernel;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\URL;
use Stancl\Tenancy\Events\SyncedResourceSaved;

trait InteractsWithTenancy
{
    public array $tenants = [];

    public string $defaultTenant = 'domainA';

    private static bool $migrated = false;

    protected string $databasePrefix = 'test_tenant_';

    protected string $domainPrefix = 'test_tenant_';

    protected bool $disableSyncMaster = true;

    public function setUpInteractsWithTenancy(): void
    {
        $databasePrefix = $this->databasePrefix;

        config(['tenancy.database.prefix' => $databasePrefix]);

        $this->tenants = empty($this->tenants) ? ['domainA', 'domainB'] : $this->tenants;

        foreach ($this->tenants as $tenant) {
            // 檢查有沒有資料庫，有的話就不建立
            $manager = (new TenantModel)->database()->manager();
            if ($manager->databaseExists($databasePrefix.$tenant)) {
                $model = TenantModel::withoutEvents(fn () => TenantModel::firstOrCreate(['id' => $tenant]));
            } else {
                $model = TenantModel::firstOrCreate(['id' => $tenant]);
            }
            $model->domains()->firstOrCreate(['domain' => "{$this->domainPrefix}$tenant"]);
        }

        $this->tearDownInteractsWithTenancy();

        if ($this->disableSyncMaster) {
            $this->disableSyncMaster();
        }

        if ($this->defaultTenant) {
            $this->switchTenant($this->defaultTenant);
        } else {
            $this->switchToCentral();
        }
    }

    public function tearDownInteractsWithTenancy(): void
    {
        if ($this->hasRefreshDatabase()) {
            if (! self::$migrated) {
                $this->artisan('tenants:migrate-fresh');
                $this->app[Kernel::class]->setArtisan(null);
                self::$migrated = true;
            }
        }
    }

    // 取消 central 和 tenant 同步功能
    public function disableSyncMaster()
    {
        Event::fake([
            SyncedResourceSaved::class,
        ]);
    }

    public function switchTenant(string $tenant): void
    {
        $model = TenantModel::find($tenant->value);
        if (! $model) {
            throw new \Exception('Tenant not found');
        }

        tenancy()->initialize($model);

        //確保 api 呼叫也可以正常切換
        config(['app.url' => "http://{$this->domainPrefix}{$tenant->value}"]);

        URL::forceRootUrl(config('app.url'));
    }

    public function switchToCentral(): void
    {
        tenancy()->end();

        config(['app.url' => 'http://localhost']);

        URL::forceRootUrl(config('app.url'));
    }

    private function hasRefreshDatabase()
    {
        $uses = array_flip(class_uses_recursive(static::class));

        return isset($uses[RefreshDatabase::class]);
    }

    private function actingAsUser(\App\Models\UserModel|array|null $user = null, ?string $guard = null)
    {
        if (! $user || is_array($user)) {
            $user = \App\Models\UserModel::factory()->create($user ?? []);
        }
        $this->actingAs($user, $guard);

        return $user;
    }
}
```

程式碼也放在[這邊](https://gist.github.com/henry11996/6643fdb6fcfb6c7da673c857333eb6ef)
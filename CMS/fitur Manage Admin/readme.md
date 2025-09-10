# Laporan: CRUD “Management Administrator” (Laravel + Filament v4.0.8)

## Ringkasan stack

* PHP 8.4.x, Laravel 12.x
* Filament 4.0.8 (`filament/*` lengkap)
* Model: pakai tabel `users` (bukan tabel baru)

---

## 1) Skema database (alter `users`)

**Buat dengan perintah:** 
```php
php artisan make:migration add_admin_fields_to_users_table --table=users
```

**Migration:** `database/migrations/20xx_xx_xx_xxxxxx_add_admin_fields_to_users_table.php`

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void {
        Schema::table('users', function (Blueprint $t) {
            $t->string('username')->unique()->after('name');
            $t->string('role')->default('editor')->after('password');    // enum string: 'super_admin', 'editor'
            $t->unsignedTinyInteger('status')->default(1)->after('role'); // 1=Aktif, 0/2=Suspended
            $t->timestamp('last_login_at')->nullable()->after('status');

            $t->unsignedBigInteger('created_by')->nullable();
            $t->unsignedBigInteger('updated_by')->nullable();
            $t->unsignedBigInteger('deleted_by')->nullable();

            $t->softDeletes();
        });
    }

    public function down(): void {
        Schema::table('users', function (Blueprint $t) {
            $t->dropColumn([
                'username','role','status','last_login_at',
                'created_by','updated_by','deleted_by','deleted_at',
            ]);
        });
    }
};
```

Jalankan:

```bash
php artisan migrate
```

---

## 2) Enums

**`app/Enums/AdminRole.php`**

```php
<?php
namespace App\Enums;

enum AdminRole: string
{
    case SuperAdmin = 'super_admin';
    case Editor     = 'editor';        // jika ingin 'admin', ubah di sini

    public function label(): string
    {
        return match($this) {
            self::SuperAdmin => 'Super Admin',
            self::Editor     => 'Editor',
        };
    }
}
```

**`app/Enums/AdminStatus.php`**

```php
<?php
namespace App\Enums;

enum AdminStatus: int
{
    case Aktif     = 1;
    case Suspended = 2;

    public function label(): string
    {
        return $this === self::Aktif ? 'Aktif' : 'Suspended';
    }
}
```

---

## 3) Trait audit (optional tapi enak)

**`app/Models/Traits/Blameable.php`**

```php
<?php
namespace App\Models\Traits;

use Illuminate\Support\Facades\Auth;

trait Blameable
{
    protected static function bootBlameable(): void
    {
        static::creating(function ($m) {
            if (Auth::check()) $m->created_by ??= Auth::id();
        });
        static::updating(function ($m) {
            if (Auth::check()) $m->updated_by = Auth::id();
        });
        static::deleting(function ($m) {
            if (method_exists($m, 'isForceDeleting') && ! $m->isForceDeleting()) {
                if (Auth::check()) $m->deleted_by = Auth::id();
                $m->saveQuietly();
            }
        });
    }
}
```

---

## 4) Listener last login

**Buat listener:**

```bash
php artisan make:listener UpdateLastLogin --event=Illuminate\\Auth\\Events\\Login
```

**`app/Listeners/UpdateLastLogin.php`**

```php
<?php
namespace App\Listeners;

use Illuminate\Auth\Events\Login;

class UpdateLastLogin
{
    public function handle(Login $event): void
    {
        $event->user->forceFill(['last_login_at' => now()])->saveQuietly();
    }
}
```

**Daftarkan di `app/Providers/EventServiceProvider.php`:**

```php
protected $listen = [
    \Illuminate\Auth\Events\Login::class => [
        \App\Listeners\UpdateLastLogin::class,
    ],
];
```

Lalu:

```bash
php artisan event:clear && php artisan event:cache
```

---

## 5) Model `User`

**`app/Models/User.php`**

```php
<?php

namespace App\Models;

use App\Enums\AdminRole;
use App\Enums\AdminStatus;
use App\Models\Traits\Blameable;
use Filament\Models\Contracts\FilamentUser;
use Filament\Models\Contracts\HasName;
use Filament\Panel;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable implements FilamentUser, HasName
{
    use HasFactory, Notifiable, SoftDeletes, Blameable;

    protected $fillable = [
        'name','username','email','password',
        'role','status','last_login_at',
        'created_by','updated_by','deleted_by',
    ];

    protected $hidden = ['password','remember_token'];

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'last_login_at'     => 'datetime',
            'password'          => 'hashed',
            'role'              => AdminRole::class,
            'status'            => AdminStatus::class,
        ];
    }

    public function canAccessPanel(Panel $panel): bool
    {
        return $this->status === AdminStatus::Aktif;
    }

    public function getFilamentName(): string
    {
        return $this->name ?: $this->username ?: (string) $this->email;
    }

    public function isSuperAdmin(): bool
    {
        return $this->role === AdminRole::SuperAdmin;
    }
}
```

---

## 6) Policy (opsional – contoh sederhana)

**`php artisan make:policy UserPolicy --model=User`**

**Isi minimal:**

```php
public function before(User $user): ?bool
{
    // Super Admin boleh semua
    if ($user->isSuperAdmin()) return true;
    return null;
}
```

---

## 7) Filament Resource

**Generate:**

```bash
php artisan make:filament-resource Admin --generate
# Title attribute: name
# Read-only view page: no
# Uses soft-deletes: yes
```

### 7a) `AdminResource`

**`app/Filament/Resources/Admins/AdminResource.php`**

```php
<?php

namespace App\Filament\Resources\Admins;

use App\Filament\Resources\Admins\Pages\{CreateAdmin, EditAdmin, ListAdmins};
use App\Filament\Resources\Admins\Schemas\AdminForm;
use App\Filament\Resources\Admins\Tables\AdminsTable;
use App\Models\User;
use BackedEnum;
use Filament\Resources\Resource;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Table;

class AdminResource extends Resource
{
    protected static ?string $model = User::class;

    protected static string|BackedEnum|null $navigationIcon = Heroicon::OutlinedUserGroup;
    protected static \UnitEnum|string|null $navigationGroup = 'CMS';
    protected static ?string $recordTitleAttribute = 'name';

    public static function form(Schema $schema): Schema
    {
        return AdminForm::configure($schema);
    }

    public static function table(Table $table): Table
    {
        return AdminsTable::configure($table);
    }

    public static function getPages(): array
    {
        return [
            'index'  => ListAdmins::route('/'),
            'create' => CreateAdmin::route('/create'),
            'edit'   => EditAdmin::route('/{record}/edit'),
        ];
    }
}
```

### 7b) Table (tanpa row actions)

**`app/Filament/Resources/Admins/Tables/AdminsTable.php`**
(kolom “Nama” jadi link ke halaman Edit; filter Role/Status; Trashed)

```php
<?php

namespace App\Filament\Resources\Admins\Tables;

use App\Enums\AdminRole;
use App\Enums\AdminStatus;
use App\Filament\Resources\Admins\AdminResource;
use App\Models\User;
use Filament\Tables;
use Filament\Tables\Table;

class AdminsTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('id')->sortable(),

                Tables\Columns\TextColumn::make('name')
                    ->label('Nama')
                    ->searchable()->sortable()
                    ->url(fn (User $record) => AdminResource::getUrl('edit', ['record' => $record])),

                Tables\Columns\TextColumn::make('email')
                    ->label('E-Mail')->searchable(),

                Tables\Columns\BadgeColumn::make('role')
                    ->label('Role')
                    ->colors([
                        'success' => fn ($s) => $s instanceof AdminRole ? $s === AdminRole::SuperAdmin : $s === 'super_admin',
                        'primary' => fn ($s) => $s instanceof AdminRole ? $s === AdminRole::Editor     : $s === 'editor',
                    ])
                    ->formatStateUsing(fn ($s) => $s instanceof AdminRole ? $s->label() : AdminRole::from((string)$s)->label()),

                Tables\Columns\BadgeColumn::make('status')
                    ->label('Status')
                    ->colors([
                        'success' => fn ($s) => $s instanceof AdminStatus ? $s === AdminStatus::Aktif     : (int)$s === 1,
                        'danger'  => fn ($s) => $s instanceof AdminStatus ? $s === AdminStatus::Suspended : (int)$s !== 1,
                    ])
                    ->formatStateUsing(fn ($s) => $s instanceof AdminStatus ? $s->label() : AdminStatus::from((int)$s)->label()),

                Tables\Columns\TextColumn::make('created_at')->label('Created At')->dateTime('j M Y H:i')->sortable(),
                Tables\Columns\TextColumn::make('last_login_at')->label('Last Login At')->dateTime('j M Y H:i')->sortable(),
            ])
            ->filters([
                Tables\Filters\SelectFilter::make('role')
                    ->label('Role')
                    ->options([
                        AdminRole::SuperAdmin->value => 'Super Admin',
                        AdminRole::Editor->value     => 'Editor',
                    ]),
                Tables\Filters\SelectFilter::make('status')
                    ->label('Status')
                    ->options([
                        AdminStatus::Aktif->value     => 'Aktif',
                        AdminStatus::Suspended->value => 'Suspended',
                    ]),
                Tables\Filters\TrashedFilter::make(),
            ]);
    }
}
```

### 7c) Form (kompatibel v4.0.8)

**`app/Filament/Resources/Admins/Schemas/AdminForm.php`**

```php
<?php

namespace App\Filament\Resources\Admins\Schemas;

use App\Enums\AdminRole;
use App\Enums\AdminStatus;
use Filament\Forms;
use Filament\Schemas\Schema;

class AdminForm
{
    public static function configure(Schema $schema): Schema
    {
        $roleOptions = collect(AdminRole::cases())->mapWithKeys(fn($c)=>[$c->value=>$c->label()])->all();
        $statusOptions = collect(AdminStatus::cases())->mapWithKeys(fn($c)=>[$c->value=>$c->label()])->all();

        $isCreate = static fn(): bool => request()->routeIs('filament.admin.resources.admins.create');

        return $schema->schema([
            Forms\Components\TextInput::make('name')->label('Nama')->required()->maxLength(255),
            Forms\Components\TextInput::make('username')->label('Username')->required()->unique(ignoreRecord: true),
            Forms\Components\TextInput::make('email')->label('E-Mail')->email()->required()->unique(ignoreRecord: true),

            Forms\Components\TextInput::make('password')
                ->label('Password')->password()->revealable()
                ->required(static fn () => $isCreate())
                ->dehydrated(static fn ($state) => filled($state))
                ->rules(static fn () => $isCreate() ? ['required','min:6'] : ['nullable','min:6']),

            Forms\Components\Select::make('role')->label('Role')->options($roleOptions)->required(),
            Forms\Components\Select::make('status')->label('Status')->options($statusOptions)->required(),
        ]);
    }
}
```

### 7d) Page actions (Delete/Restore di halaman Edit)

**`app/Filament/Resources/Admins/Pages/EditAdmin.php`**

```php
protected function getHeaderActions(): array
{
    return [
        \Filament\Actions\DeleteAction::make(),
        \Filament\Actions\ForceDeleteAction::make(),
        \Filament\Actions\RestoreAction::make(),
    ];
}
```

---

## 8) Seeder Super Admin

**`database/seeders/SuperAdminSeeder.php`**

```php
<?php

namespace Database\Seeders;

use App\Enums\AdminRole;
use App\Enums\AdminStatus;
use App\Models\User;
use Illuminate\Database\Seeder;

class SuperAdminSeeder extends Seeder
{
    public function run(): void
    {
        $admin = User::updateOrCreate(
            ['email' => 'superadmin@lokatani.id'],
            [
                'name'     => 'Super Admin',
                'username' => 'superadmin',
                'password' => 'password',             // akan di-hash otomatis oleh casts
                'role'     => AdminRole::SuperAdmin,
                'status'   => AdminStatus::Aktif,
            ]
        );

        if (! $admin->created_by) {
            $admin->forceFill(['created_by' => $admin->id])->saveQuietly();
        }
    }
}
```

Jalankan:

```bash
php artisan db:seed --class=SuperAdminSeeder
```

---

## 9) Bersih-bersih cache (sering menyelamatkan!)

```bash
composer dump-autoload -o
php artisan optimize:clear
php artisan event:cache
```

---

## 10) Cara pakai di panel

* Menu: **CMS → Management Administrator**
* **List**: klik **Nama** untuk masuk **Edit**
* **Create**: `/admin/admins/create`
* **Delete/Restore**: tombol di header halaman **Edit**
* Password: saat Edit boleh kosong (tidak berubah); saat Create wajib (min 6)

---

## Troubleshooting cepat

* **`EditAction not found`** → hilangkan `Tables\Actions\*` di table, pakai link di kolom nama + header actions (seperti di atas).
* **`Section not found`** → jangan pakai `Forms\Components\Section` (tidak ada di v4.0.8).
* **Validator `nullable|min`** → gunakan `->rules(['nullable','min:6'])`, jangan pipe di satu string.
* **Closure `$op` unresolvable** → jangan pakai parameter di closure `required()`. Deteksi create pakai `request()->routeIs(...)`.
* **Enum error `from()`** → jika `$state` sudah enum, jangan `from()` lagi; cek instance‐of dulu.

---

## Cheat-sheet perintah

```bash
# generate resource
php artisan make:filament-resource Admin --generate

# listener last login
php artisan make:listener UpdateLastLogin --event=Illuminate\\Auth\\Events\\Login

# seeder
php artisan make:seeder SuperAdminSeeder
php artisan db:seed --class=SuperAdminSeeder

# bersih cache
composer dump-autoload -o
php artisan optimize:clear
```

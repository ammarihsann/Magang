Berikut **dokumen ringkas** (runbook) supaya kamu bisa membangun fitur **Manage Peserta Pelatihan** dengan cepat di masa depan. Fokus: Laravel 12 + Filament v4 (schema-based API), enum role, soft delete aware, validasi unik email (abaikan soft-deleted), dan batasan Editor vs Super Admin.

---

# Plan & Acceptances

**Scope**

* CRUD Peserta (soft delete).
* Editor: hanya boleh **edit/hapus/restore data yang dia buat** (`created_by = user.id`).
* Super Admin: full akses + bulk actions.
* Form mematuhi aturan: email unik (abaikan soft-deleted), Date/Time format “d, F Y H\:i”, timezone Asia/Jakarta.
* Generate password acak 12 char lewat toggle di field `new_password`.
* Tabel peserta soft-delete aware (TrashedFilter), eager load `agenda`.
* Aksi sertifikat (generate/download) disiapkan hook (opsional).

**Acceptances**

* Editor tidak melihat bulk actions & tombol Delete di tabel.
* Editor hanya bisa edit/restore/hapus **miliknya sendiri** di halaman Edit.
* Super Admin melihat semua aksi (record & bulk).
* Form “E-Mail” menolak duplikasi jika record lain **belum** dihapus (ignore soft-deleted).
* Toggle “Generate 12 karakter” mengisi `new_password` tanpa menyimpan langsung.
* Kolom tanggal tampil “d, F Y H\:i” (Asia/Jakarta).

---

# Code (ringkasan file & potongan penting)

> Kamu sudah punya sebagian besar file. Ini daftar **inti** + cuplikan yang wajib ada/cek.

## 1) Enum Role

**app/Enums/AdminRole.php**

```php
<?php
namespace App\Enums;

enum AdminRole: string {
    case SuperAdmin = 'super_admin';
    case Editor = 'editor';
}
```

## 2) Model Participant

**app/Models/Participant.php** (cuplikan penting)

```php
<?php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use App\Traits\Blameable;

class Participant extends Model
{
    use SoftDeletes, Blameable;

    protected $fillable = [
        'agenda_id','full_name','email','phone','domicile','occupation',
        'birthdate','instagram','referral','attendance','status',
        'certificate_published_at','password','created_by','updated_by','deleted_by',
    ];

    protected $casts = [
        'attendance' => 'boolean',
        'certificate_published_at' => 'datetime',
        'birthdate' => 'date',
    ];

    public function agenda() { return $this->belongsTo(Agenda::class); }

    // Mutator hash password (Laravel 12)
    public function setPasswordAttribute($value): void
    {
        if ($value) $this->attributes['password'] = bcrypt($value);
    }
}
```

## 3) Policy (kunci role)

**app/Policies/ParticipantPolicy.php**

```php
<?php
namespace App\Policies;

use App\Enums\AdminRole;
use App\Models\Participant;
use App\Models\User;

class ParticipantPolicy
{
    public function viewAny(User $u): bool { return true; }
    public function view(User $u, Participant $p): bool { return true; }
    public function create(User $u): bool { return true; }

    public function update(User $u, Participant $p): bool
    {
        return $u->role === AdminRole::SuperAdmin || ( $u->role === AdminRole::Editor && (int)$p->created_by === (int)$u->id );
    }
    public function delete(User $u, Participant $p): bool
    {
        return $u->role === AdminRole::SuperAdmin || ( $u->role === AdminRole::Editor && (int)$p->created_by === (int)$u->id );
    }
    public function restore(User $u, Participant $p): bool
    {
        return $u->role === AdminRole::SuperAdmin || ( $u->role === AdminRole::Editor && (int)$p->created_by === (int)$u->id );
    }
    public function forceDelete(User $u, Participant $p): bool
    {
        return $u->role === AdminRole::SuperAdmin;
    }
}
```

**AuthServiceProvider** (angkat Super Admin)

```php
Gate::before(fn ($user) => $user->role === \App\Enums\AdminRole::SuperAdmin ? true : null);
```

## 4) Resource & Table

**app/Filament/Resources/Participants/ParticipantResource.php** (schema-based)

```php
public static function form(\Filament\Schemas\Schema $schema): \Filament\Schemas\Schema
{
    return \App\Filament\Resources\Participants\Schemas\ParticipantForm::configure($schema);
}
```

**app/Filament/Resources/Participants/Tables/ParticipantsTable.php** (intinya editor tak punya bulk & delete di tabel)

```php
$isSuperAdmin = auth()->user()?->role === \App\Enums\AdminRole::SuperAdmin;

return $table
    ->query(fn () => \App\Models\Participant::query()
        ->withoutGlobalScopes([\Illuminate\Database\Eloquent\SoftDeletingScope::class])
        ->with(['agenda'])
    )
    ->columns([/* ID, Nama, Agenda, Email, Phone, Attendance, Certificate, CreatedAt */])
    ->filters([
        \Filament\Tables\Filters\TrashedFilter::make()->label('Terhapus'),
        \Filament\Tables\Filters\SelectFilter::make('status')->label('Status')->options([
            \App\Enums\RecordStatus::Active->value => 'Aktif',
            \App\Enums\RecordStatus::Draft->value => 'Draft',
        ]),
    ])
    ->recordActions([
        \Filament\Actions\DeleteAction::make()->label('Hapus')->visible(fn() => $isSuperAdmin),
        \Filament\Actions\RestoreAction::make()->label('Pulihkan')->visible(fn ($r) => auth()->user()->can('restore', $r)),
        \Filament\Actions\ForceDeleteAction::make()->label('Hapus Permanen')->visible(fn ($r) => auth()->user()->can('forceDelete', $r)),
    ])
    ->toolbarActions($isSuperAdmin ? [
        \Filament\Actions\BulkActionGroup::make([
            \Filament\Actions\DeleteBulkAction::make()->label('Hapus'),
            \Filament\Actions\ForceDeleteBulkAction::make()->label('Hapus Permanen'),
            \Filament\Actions\RestoreBulkAction::make()->label('Pulihkan'),
        ]),
    ] : []);
```

## 5) Form (Schema API + toggle generate password)

**app/Filament/Resources/Participants/Schemas/ParticipantForm.php** (potongan unik email & toggle)

```php
use Filament\Schemas\Schema;
use Filament\Forms\Components\{TextInput, Select, DatePicker, DateTimePicker, Toggle, Textarea};
use Filament\Schemas\Components\Utilities\Set; // penting untuk Schema API
use Illuminate\Validation\Rules\Unique;

public static function configure(Schema $schema): Schema
{
    $tz = 'Asia/Jakarta';

    return $schema->schema([
        Select::make('agenda_id')->label('Agenda Pelatihan')->required()
            ->options(fn () => \Cache::remember('agenda_options', 600,
                fn () => \App\Models\Agenda::query()->orderBy('title')->pluck('title','id')->all()
            ))
            ->searchable()->preload(),

        TextInput::make('full_name')->label('Nama Lengkap')->required()->maxLength(191),

        TextInput::make('email')->label('E-Mail')->email()->required()->maxLength(191)
            ->unique(
                table: 'participants',
                column: 'email',
                ignoreRecord: true,
                modifyRuleUsing: fn (Unique $rule) => $rule->whereNull('deleted_at')
            ),

        // ... phone, domicile, occupation, birthdate, instagram, referral, attendance, description ...

        DateTimePicker::make('certificate_published_at')
            ->label('Sertifikat Terbit')->timezone($tz)->displayFormat('d, F Y H:i')
            ->native(false)->disabled()->dehydrated(false),

        TextInput::make('new_password')->label('Password Baru')->password()->revealable()
            ->minLength(6)->maxLength(64)->dehydrated(false)->visibleOn('edit'),

        Toggle::make('generate_random_password')->label('Generate 12 karakter')
            ->helperText('Klik untuk mengisi password acak A–Z, a–z, 0–9 (12 char).')
            ->live()
            ->afterStateUpdated(function (bool $state, Set $set) {
                if ($state) {
                    $set('new_password', \Str::password(12, symbols: false));
                    $set('generate_random_password', false);
                }
            })
            ->dehydrated(false)->visibleOn('edit'),
    ]);
}
```

## 6) Halaman Edit (memindah `new_password` → `password`)

**app/Filament/Resources/Participants/Pages/EditParticipant.php**

```php
protected function mutateFormDataBeforeSave(array $data): array
{
    if (!empty($data['new_password'])) $data['password'] = $data['new_password'];
    unset($data['new_password']);
    return $data;
}
```

## 7) Migration Peserta (inti)

* `participants` punya FK `agenda_id`, soft deletes, index email unik (optional di DB level, tetapi validasi sudah handle soft-deleted).
* Password string (hash via mutator model).

> Catatan: untuk **email unik yang abaikan soft-deleted**, cukup di **validasi**. Jika ingin constraint DB unik, butuh partial index (tidak native di MySQL) — jadi **jangan** pakai unique index DB default pada `email`.

---

# Tests (Pest) — ringkas

**tests/Feature/Participants/ParticipantModelTest.php**

* “auto hash password”
* “email unik abaikan soft-deleted”

```php
it('hashes password', function () {
    $p = \App\Models\Participant::factory()->create(['password' => 'secret123']);
    expect($p->password)->not->toBe('secret123')->and(password_verify('secret123', $p->password))->toBeTrue();
});

it('validates unique email ignoring soft-deleted', function () {
    $email = 'u@example.com';
    \App\Models\Participant::factory()->create(['email' => $email]);
    $deleted = \App\Models\Participant::factory()->create(['email' => 'x@example.com']);
    $deleted->delete();

    $data = ['email' => $email];
    $v = \Validator::make($data, [
        'email' => ['required','email', \Illuminate\Validation\Rule::unique('participants','email')->whereNull('deleted_at')],
    ]);
    expect($v->fails())->toBeTrue();

    $v2 = \Validator::make(['email' => 'x@example.com'], [
        'email' => ['required','email', \Illuminate\Validation\Rule::unique('participants','email')->whereNull('deleted_at')],
    ]);
    expect($v2->fails())->toBeFalse();
});
```

**tests/Feature/Participants/ParticipantPolicyTest.php**

* Editor boleh edit/restore miliknya, tidak untuk orang lain.
* Super Admin full.

```php
uses(\Illuminate\Foundation\Testing\RefreshDatabase::class);

it('editor can update/restore own participant only', function () {
    $editor = \App\Models\User::factory()->create(['role' => \App\Enums\AdminRole::Editor]);
    $mine = \App\Models\Participant::factory()->create(['created_by' => $editor->id]);
    $other = \App\Models\Participant::factory()->create();

    expect(\Gate::forUser($editor)->allows('update', $mine))->toBeTrue();
    expect(\Gate::forUser($editor)->allows('update', $other))->toBeFalse();

    $mine->delete();
    $other->delete();
    expect(\Gate::forUser($editor)->allows('restore', $mine))->toBeTrue();
    expect(\Gate::forUser($editor)->allows('restore', $other))->toBeFalse();
});

it('super admin can do everything', function () {
    $sa = \App\Models\User::factory()->create(['role' => \App\Enums\AdminRole::SuperAdmin]);
    $p = \App\Models\Participant::factory()->create();

    foreach (['viewAny','view','create','update','delete','restore','forceDelete'] as $ability) {
        expect(\Gate::forUser($sa)->allows($ability, $p))->toBeTrue();
    }
});
```

---

# Next Steps (Runbook)

**1) Composer & cache**

```bash
composer install
php artisan optimize:clear
php artisan storage:link
```

**2) Migrate & seed**

```bash
php artisan migrate --seed
```

**3) Cek policy terdaftar**

* `AuthServiceProvider::$policies[Participant::class] = ParticipantPolicy::class;`
* `Gate::before` untuk Super Admin aktif.

**4) Uji cepat di UI**

* Login sebagai **Editor** → pastikan:

  * Tidak ada **bulk actions** & checkbox di tabel.
  * Tidak ada tombol **Delete** di tabel; **Delete** hanya tersedia di halaman **Edit** miliknya.
  * Restore hanya miliknya.
* Login sebagai **Super Admin** → semua aksi tersedia.

**5) Git flow singkat**

```bash
git checkout -b feature/manage-peserta
# kerja...
git fetch origin
git rebase origin/development   # atau merge jika tim melarang rebase
git push --force-with-lease
# buka PR -> development
```

**6) Pitfall yang sering muncul & solusinya**

* **Class ‘Section/Grid’ not found (Filament v4.0.x)** → gunakan **Schema API tanpa Section/Grid**, pakai komponen dasar.
* **Unique email salah tipe closure** → gunakan `use Illuminate\Validation\Rules\Unique;` dan `fn (Unique $rule)`.
* **Set type mismatch** → untuk Schema API pakai `use Filament\Schemas\Components\Utilities\Set;` (bukan `Forms\Set`).
* **Editor masih bisa Delete di tabel** → pastikan `ParticipantsTable` men-guard visibilitas Delete dengan `$isSuperAdmin`.

Kalau kamu mau, kita bisa tambahkan template **stubs** (artisan make\:filament-resource + auto-inject potongan di atas) agar implementasi berikutnya tinggal 1–2 menit kerja.

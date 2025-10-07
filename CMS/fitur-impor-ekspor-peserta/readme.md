# 📘 Dokumentasi Fitur Impor & Ekspor Peserta (Laravel 12 + Filament v4)

Dokumen ini menjelaskan cara kerja dan kode penuh untuk:

* **Impor Peserta** dari file **Excel/CSV** (xls/xlsx/csv) dengan password default `12345678`.
* **Ekspor Peserta** ke **Excel (.xlsx)**.
* Integrasi dengan **Filament v4** (Livewire 3), **SPA-safe** (bypass SPA untuk download), soft delete aware, dan performa aman (chunk).

---

## Plan & Acceptances

* [x] **Impor Excel/CSV**:

  * Tombol **Impor Excel** di halaman **Peserta** (Filament `ListParticipants`).
  * Menerima `xls/xlsx/csv` (maks 10MB), disimpan sementara di `storage/app/imports/participants`, lalu dihapus setelah proses.
  * Kolom **wajib**: `full_name`, `email`; **opsional**: `phone`, `domicile`, `occupation`, `birthdate`, `instagram`, `referral` (+ alias Indonesia).
  * Validasi email **unik** (mengabaikan soft deletes).
  * Password default **`12345678`** (di-hash).
  * Mendukung `birthdate` ISO (`YYYY-MM-DD`), `dd/mm/YYYY`, `dd-mm-YYYY`, Excel serial.

* [x] **Ekspor Excel**:

  * Tombol **Ekspor Excel** di halaman **Peserta**.
  * Menghasilkan `peserta-export.xlsx` berisi peserta **non-trashed** (soft delete aware).
  * Kolom: `ID, Nama, E-Mail, No. HP, Domisili, Pekerjaan, Tgl Lahir, Instagram, Sumber Info, Dibuat`.
  * Download **dibuka di tab baru** untuk bypass SPA Filament.

* [x] **UI Indonesia**: label tombol & notifikasi berbahasa Indonesia.

* [x] **Kinerja**: query export pakai `FromQuery` (chunk), impor aman tanpa N+1.

* [x] **Ops**: Composer, route di **AdminPanelProvider**, runbook, dan tes Pest.

---

## Ops / Instalasi

```bash
# 1) Install Laravel Excel (kompatibel PHP 8.4, Laravel 12)
composer require maatwebsite/excel:"^3.1.67" --with-all-dependencies

# 2) (Opsional) Publish config
php artisan vendor:publish --provider="Maatwebsite\Excel\ExcelServiceProvider" --tag=config

# 3) (Opsional untuk impor besar - pakai queue)
php artisan queue:table && php artisan migrate
# .env -> QUEUE_CONNECTION=database

# 4) Umum
php artisan migrate
php artisan storage:link
php artisan optimize:clear
```

**.env penting (contoh minimal):**

```
APP_TIMEZONE=Asia/Jakarta
QUEUE_CONNECTION=database
```

---

## Code

### 1) Exporter (Excel)

**File:** `app/Exports/ParticipantsExport.php`

```php
<?php

namespace App\Exports;

use App\Models\Participant;
use Illuminate\Database\Eloquent\Builder;
use Maatwebsite\Excel\Concerns\FromQuery;
use Maatwebsite\Excel\Concerns\WithHeadings;
use Maatwebsite\Excel\Concerns\WithMapping;
use Maatwebsite\Excel\Concerns\ShouldAutoSize;
use Maatwebsite\Excel\Concerns\WithColumnFormatting;
use PhpOffice\PhpSpreadsheet\Style\NumberFormat;

class ParticipantsExport implements FromQuery, WithHeadings, WithMapping, ShouldAutoSize, WithColumnFormatting
{
    public function query(): Builder
    {
        return Participant::query()
            ->select([
                'id','full_name','email','phone','domicile','occupation',
                'birthdate','instagram','referral','created_at',
            ])
            ->whereNull('deleted_at')
            ->orderBy('id');
    }

    public function headings(): array
    {
        return [
            'ID',
            'Nama',
            'E-Mail',
            'No. HP',
            'Domisili',
            'Pekerjaan',
            'Tgl Lahir',
            'Instagram',
            'Sumber Info',
            'Dibuat',
        ];
    }

    /**
     * @param \App\Models\Participant $p
     */
    public function map($p): array
    {
        return [
            $p->id,
            $p->full_name,
            $p->email,
            $p->phone,
            $p->domicile,
            $p->occupation,
            optional($p->birthdate)->toDateString(),
            $p->instagram,
            $p->referral,
            $p->created_at?->timezone('Asia/Jakarta')->format('Y-m-d H:i:s'),
        ];
    }

    public function columnFormats(): array
    {
        return [
            'C' => NumberFormat::FORMAT_TEXT, // email
            'D' => NumberFormat::FORMAT_TEXT, // phone (hindari scientific)
        ];
    }
}
```

**File:** `app/Http/Controllers/Admin/ParticipantExportController.php`

```php
<?php

namespace App\Http\Controllers\Admin;

use Illuminate\Routing\Controller as BaseController;
use Maatwebsite\Excel\Facades\Excel;
use App\Exports\ParticipantsExport;

class ParticipantExportController extends BaseController
{
    public function exportXlsx()
    {
        return Excel::download(new ParticipantsExport(), 'peserta-export.xlsx');
    }
}
```

### 2) Importer (Excel/CSV) – **mendukung kolom opsional + alias Indonesia**

**File:** `app/Imports/ParticipantsImport.php`

```php
<?php

namespace App\Imports;

use App\Models\Participant;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rule;
use Maatwebsite\Excel\Concerns\ToModel;
use Maatwebsite\Excel\Concerns\WithHeadingRow;
use Maatwebsite\Excel\Concerns\WithValidation;
use Maatwebsite\Excel\Concerns\SkipsOnFailure;
use Maatwebsite\Excel\Validators\Failure;
use Maatwebsite\Excel\Concerns\SkipsFailures;
use PhpOffice\PhpSpreadsheet\Shared\Date as ExcelDate;
use Carbon\Carbon;

class ParticipantsImport implements ToModel, WithHeadingRow, WithValidation, SkipsOnFailure
{
    use SkipsFailures;

    /**
     * @param array<string, mixed> $row
     */
    public function model(array $row)
    {
        // Wajib
        $fullName = $row['full_name'] ?? $row['nama'] ?? null;
        $email    = $row['email'] ?? null;

        // Opsional + alias Indonesia
        $phone      = $row['phone'] ?? $row['no_hp'] ?? null;
        $domicile   = $row['domicile'] ?? $row['domisili'] ?? null;
        $occupation = $row['occupation'] ?? $row['pekerjaan'] ?? null;
        $instagram  = $row['instagram'] ?? null;
        $referral   = $row['referral'] ?? $row['sumber_informasi'] ?? $row['sumber informasi'] ?? null;

        $birthdateInput = $row['birthdate'] ?? $row['tanggal_lahir'] ?? $row['tanggal lahir'] ?? null;
        $birthdate = $this->parseBirthdate($birthdateInput);

        if (! $email || ! $fullName) {
            return null;
        }

        $exists = Participant::query()
            ->where('email', mb_strtolower(trim($email)))
            ->whereNull('deleted_at')
            ->exists();

        if ($exists) {
            return null;
        }

        return new Participant([
            'full_name'  => $fullName,
            'email'      => $email,
            'phone'      => $phone,
            'domicile'   => $domicile,
            'occupation' => $occupation,
            'birthdate'  => $birthdate,
            'instagram'  => $instagram,
            'referral'   => $referral,
            'password'   => Hash::make('12345678'),
        ]);
    }

    public function rules(): array
    {
        return [
            '*.full_name' => ['required', 'string', 'min:3'],
            '*.email'     => [
                'required', 'email',
                Rule::unique('participants', 'email')->whereNull('deleted_at'),
            ],
            '*.phone'       => ['nullable', 'string', 'max:32'],
            '*.domicile'    => ['nullable', 'string', 'max:255'],
            '*.occupation'  => ['nullable', 'string', 'max:255'],
            '*.instagram'   => ['nullable', 'string', 'max:255'],
            '*.referral'    => ['nullable', 'string', 'max:255'],
            '*.birthdate'   => ['nullable', 'date'],
        ];
    }

    public function customValidationMessages()
    {
        return [
            '*.full_name.required' => 'Nama wajib diisi.',
            '*.email.required'     => 'E-mail wajib diisi.',
            '*.email.email'        => 'Format e-mail tidak valid.',
            '*.email.unique'       => 'E-mail sudah terdaftar.',
            '*.birthdate.date'     => 'Format tanggal lahir tidak valid.',
        ];
    }

    protected function parseBirthdate($value): ?string
    {
        if ($value === null || $value === '') {
            return null;
        }

        // Excel serial
        if (is_numeric($value)) {
            try {
                $dt = ExcelDate::excelToDateTimeObject((float) $value);
                return Carbon::instance($dt)->toDateString();
            } catch (\Throwable) {
                return null;
            }
        }

        $str = trim((string) $value);

        // ISO / fleksibel
        try {
            return Carbon::parse($str)->toDateString();
        } catch (\Throwable) {}

        foreach (['d/m/Y', 'd-m-Y', 'd.m.Y'] as $fmt) {
            try {
                return Carbon::createFromFormat($fmt, $str)->toDateString();
            } catch (\Throwable) {}
        }

        return null;
    }
}
```

### 3) Routes Panel (Filament SPA-safe)

**File:** `app/Providers/Filament/AdminPanelProvider.php`

> Route panel dibuat **di provider** supaya otomatis ikut prefix `/admin` & middleware panel.

```php
<?php

namespace App\Providers\Filament;

use Filament\Http\Middleware\Authenticate;
use Filament\Http\Middleware\AuthenticateSession;
use Filament\Http\Middleware\DisableBladeIconComponents;
use Filament\Http\Middleware\DispatchServingFilamentEvent;
use Filament\Pages\Dashboard;
use Filament\Panel;
use Filament\PanelProvider;
use Filament\Support\Colors\Color;
use Filament\Support\Enums\Width;
use Filament\Widgets\AccountWidget;
use Filament\Widgets\FilamentInfoWidget;
use Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse;
use Illuminate\Cookie\Middleware\EncryptCookies;
use Illuminate\Foundation\Http\Middleware\VerifyCsrfToken;
use Illuminate\Routing\Middleware\SubstituteBindings;
use Illuminate\Session\Middleware\StartSession;
use Illuminate\View\Middleware\ShareErrorsFromSession;
use Illuminate\Support\Facades\Route;

use App\Http\Controllers\Admin\ParticipantExportController;

class AdminPanelProvider extends PanelProvider
{
    public function panel(Panel $panel): Panel
    {
        return $panel
            ->default()
            ->id('admin')
            ->path('admin')
            ->login()
            ->spa()
            ->brandName('Lokatani CMS')
            ->maxContentWidth(Width::Full)
            ->colors(['primary' => Color::Indigo])
            ->discoverResources(in: app_path('Filament/Resources'), for: 'App\\Filament\\Resources')
            ->discoverPages(in: app_path('Filament/Pages'), for: 'App\\Filament\\Pages')
            ->pages([Dashboard::class])
            ->discoverWidgets(in: app_path('Filament/Widgets'), for: 'App\\Filament\\Widgets')
            ->widgets([AccountWidget::class, FilamentInfoWidget::class])
            ->middleware([
                EncryptCookies::class,
                AddQueuedCookiesToResponse::class,
                StartSession::class,
                AuthenticateSession::class,
                ShareErrorsFromSession::class,
                VerifyCsrfToken::class,
                SubstituteBindings::class,
                DisableBladeIconComponents::class,
                DispatchServingFilamentEvent::class,
            ])
            ->authMiddleware([Authenticate::class])
            ->routes(function () {
                // Ekspor Excel
                Route::get('participants/export.xlsx', [ParticipantExportController::class, 'exportXlsx'])
                    ->name('participants.export.xlsx'); // => filament.admin.participants.export.xlsx
            });
    }
}
```

### 4) Filament List Page – Tambah tombol Create, Impor, Ekspor

**File:** `app/Filament/Resources/Participants/Pages/ListParticipants.php`

```php
<?php

namespace App\Filament\Resources\Participants\Pages;

use App\Filament\Resources\Participants\ParticipantResource;
use App\Imports\ParticipantsImport;
use Filament\Actions;
use Filament\Forms;
use Filament\Notifications\Notification;
use Filament\Resources\Pages\ListRecords;
use Illuminate\Support\Facades\Storage;
use Maatwebsite\Excel\Facades\Excel;

class ListParticipants extends ListRecords
{
    protected static string $resource = ParticipantResource::class;

    protected function getHeaderActions(): array
    {
        return [
            // ✅ Tombol Buat Peserta
            Actions\CreateAction::make()
                ->label('Buat Peserta')
                ->icon('heroicon-o-plus'),

            // ✅ Ekspor Excel (bypass SPA -> open in new tab)
            Actions\Action::make('export_excel')
                ->label('Ekspor Excel')
                ->icon('heroicon-o-arrow-down-tray')
                ->url(route('filament.admin.participants.export.xlsx'))
                ->openUrlInNewTab(true),

            // ✅ Impor Excel/CSV
            Actions\Action::make('import_excel')
                ->label('Impor Excel')
                ->icon('heroicon-o-arrow-up-tray')
                ->modalHeading('Impor Peserta dari Excel/CSV')
                ->modalSubmitActionLabel('Impor')
                ->form([
                    Forms\Components\FileUpload::make('file')
                        ->label('File (.xls/.xlsx/.csv)')
                        ->acceptedFileTypes([
                            'application/vnd.ms-excel',
                            'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
                            'text/csv', 'text/plain', 'application/csv', 'text/comma-separated-values',
                        ])
                        ->maxSize(10240) // 10MB
                        ->directory('imports/participants')
                        ->disk('local')
                        ->visibility('private')
                        ->required(),
                ])
                ->action(function (array $data) {
                    $path = $data['file'];
                    $fullPath = Storage::disk('local')->path($path);

                    try {
                        Excel::import(new ParticipantsImport, $fullPath);
                        Storage::disk('local')->delete($path);

                        Notification::make()
                            ->title('Impor berhasil')
                            ->body('Data peserta telah diproses. Password default: 12345678.')
                            ->success()
                            ->send();
                    } catch (\Throwable $e) {
                        Notification::make()
                            ->title('Gagal impor')
                            ->body('Terjadi kesalahan saat memproses file: ' . $e->getMessage())
                            ->danger()
                            ->send();
                    }
                }),
        ];
    }
}
```

> Pastikan `ParticipantResource::getPages()` sudah mengandung `create` page:

```php
public static function getPages(): array
{
    return [
        'index'  => Pages\ListParticipants::route('/'),
        'create' => Pages\CreateParticipant::route('/create'),
        'edit'   => Pages\EditParticipant::route('/{record}/edit'),
    ];
}
```

### 5) Model (referensi, jika diperlukan)

**File:** `app/Models/Participant.php` (yang kamu gunakan sebelumnya sudah OK)

* `password` cast ke `hashed`
* enum `status` (RecordStatus)
* soft deletes & trait `Blameable`
* normalisasi email ke lowercase

---

## Tests (Pest)

**File:** `tests/Feature/ParticipantsImportTest.php`

```php
<?php

use App\Imports\ParticipantsImport;
use App\Models\Participant;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Storage;
use Maatwebsite\Excel\Facades\Excel;

uses(RefreshDatabase::class);

it('imports participants with optional fields and default password', function () {
    Storage::fake('local');

    $csv = "full_name,email,phone,domicile,occupation,birthdate,instagram,referral\n".
           "Opt User,opt.user@example.com,08123,Depok,Pelajar,17/05/2003,@optuser,Teman\n";

    $path = 'imports/participants/opt.csv';
    Storage::disk('local')->put($path, $csv);
    $fullPath = Storage::disk('local')->path($path);

    Excel::import(new ParticipantsImport, $fullPath);

    $this->assertDatabaseHas('participants', [
        'email'     => 'opt.user@example.com',
        'domicile'  => 'Depok',
        'occupation'=> 'Pelajar',
        'instagram' => '@optuser',
        'referral'  => 'Teman',
        'birthdate' => '2003-05-17',
    ]);
});
```

**File:** `tests/Feature/ParticipantsExportTest.php`

```php
<?php

use Illuminate\Foundation\Testing\RefreshDatabase;
use App\Models\User;
use App\Models\Participant;

uses(RefreshDatabase::class);

it('exports participants to xlsx', function () {
    $admin = User::factory()->create();
    $this->actingAs($admin);

    Participant::factory()->count(2)->create();

    $resp = $this->get(route('filament.admin.participants.export.xlsx'));
    $resp->assertOk();
    $resp->assertHeader('content-type', 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet');
});
```

---

## N+1 & Kinerja

* **Impor**: Tidak ada relasi yang di-load → aman dari N+1.
* **Ekspor**: `FromQuery` memecah data per chunk internal (oleh Laravel Excel) → aman untuk ribuan baris.
* **Optional**: Untuk impor besar, gunakan queue:

  ```php
  Excel::queueImport(new ParticipantsImport, $fullPath)->allOnQueue('default');
  # .env: QUEUE_CONNECTION=database
  # jalankan worker: php artisan queue:work
  ```

---

## Runbook

1. Install package (jika belum):
   `composer require maatwebsite/excel:"^3.1" --with-all-dependencies`

2. Tempel file:

   * `app/Exports/ParticipantsExport.php`
   * `app/Imports/ParticipantsImport.php`
   * `app/Http/Controllers/Admin/ParticipantExportController.php`
   * Update `app/Providers/Filament/AdminPanelProvider.php` (routes panel)
   * Update `app/Filament/Resources/Participants/Pages/ListParticipants.php` (header actions)

3. Clear cache:
   `php artisan optimize:clear`

4. Buka **Admin → Peserta**:

   * **Buat Peserta** → form create
   * **Impor Excel** → upload `.xls/.xlsx/.csv`
   * **Ekspor Excel** → file `peserta-export.xlsx` terunduh (dibuka di tab baru)

---

# Dokumentasi Fitur **Agenda Peserta** (Laravel 12 + Filament v4/Schemas)

Fitur ini menambahkan manajemen “Agenda Peserta” dengan aturan:

* Setiap **peserta (participant)** hanya boleh **1×** untuk **satu agenda** (kombinasi `(participant_id, agenda_id)` harus unik, soft-deletes aware).
* Form **Create/Edit**:

  * Email Peserta (dropdown dari `participants`)
  * Agenda Pelatihan (dropdown dari `agendas`)
  * Toggle “Tandai Hadir Sekarang” → mengisi `attended_at` otomatis.
  * Toggle “Rilis Sertifikat” → mewajibkan `certificate_number` & `certificate_published_at`.
  * Status, Deskripsi.
* **Validasi duplikasi** dilakukan di **Pages** (Create & Edit) agar error ditempel ke field yang benar (`data.agenda_id`) + toast merah.
* Tabel daftar memuat kolom: ID, E-Mail, Agenda, Waktu Hadir, Sertifikat, No. Sertifikat, Dibuat. Ada filter Trash & bulk actions.

> Zona waktu dan format tanggal: `Asia/Jakarta`, `d, F Y H:i`.

---

## Plan & Acceptances

* [x] Migration + unique index `(participant_id, agenda_id)` (soft deletes ada).
* [x] Model `AgendaPeserta` (fillable, casts, relasi).
* [x] Filament **Resource** memakai `Filament\Schemas\Schema` dan `Filament\Tables\Table`.
* [x] **Form** (Schemas) dengan Select peserta & agenda, toggle hadir, toggle rilis sertifikat.
* [x] **Table** dengan eager load (`with(['participant','agenda'])`) untuk cegah N+1.
* [x] Validasi duplikasi dipindah ke **beforeCreate/beforeSave** + error ke `data.agenda_id` dan toast merah.
* [x] Soft-delete aware list & binding.

---

## Kode

> Gunakan **path persis** di bawah ini. Timpa file lama jika sudah ada.

### 1) Migration (jika belum ada)

**database/migrations/2025_10_07_000000_create_agenda_pesertas_table.php**

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('agenda_pesertas', function (Blueprint $table) {
            $table->id();
            $table->foreignId('participant_id')->constrained('participants')->cascadeOnUpdate()->restrictOnDelete();
            $table->foreignId('agenda_id')->constrained('agendas')->cascadeOnUpdate()->restrictOnDelete();

            $table->timestamp('attended_at')->nullable();
            $table->timestamp('certificate_published_at')->nullable();
            $table->string('certificate_number')->nullable();
            $table->tinyInteger('status')->default(1);
            $table->text('description')->nullable();

            $table->timestamps();
            $table->softDeletes();

            $table->unique(['participant_id', 'agenda_id'], 'uniq_participant_agenda');
            $table->index('certificate_number');
            $table->index('status');
            $table->index('created_at');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('agenda_pesertas');
    }
};
```

### 2) Model

**app/Models/AgendaPeserta.php**

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class AgendaPeserta extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'participant_id',
        'agenda_id',
        'attended_at',
        'certificate_published_at',
        'certificate_number',
        'status',
        'description',
    ];

    protected $casts = [
        'attended_at' => 'datetime',
        'certificate_published_at' => 'datetime',
        'status' => 'integer',
    ];

    public function participant(): BelongsTo
    {
        return $this->belongsTo(Participant::class);
    }

    public function agenda(): BelongsTo
    {
        return $this->belongsTo(Agenda::class);
    }

    // Untuk judul record di UI
    public function getLabelAttribute(): string
    {
        $email = $this->participant?->email ?? '-';
        $title = $this->agenda?->title ?? '-';
        return "{$email} — {$title}";
    }
}
```

### 3) Resource

**app/Filament/Resources/AgendaPesertas/AgendaPesertaResource.php**

```php
<?php

namespace App\Filament\Resources\AgendaPesertas;

use App\Filament\Resources\AgendaPesertas\Pages\CreateAgendaPeserta;
use App\Filament\Resources\AgendaPesertas\Pages\EditAgendaPeserta;
use App\Filament\Resources\AgendaPesertas\Pages\ListAgendaPesertas;
use App\Filament\Resources\AgendaPesertas\Schemas\AgendaPesertaForm;
use App\Filament\Resources\AgendaPesertas\Tables\AgendaPesertasTable;
use App\Models\AgendaPeserta;
use BackedEnum;
use Filament\Resources\Resource;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Table;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\SoftDeletingScope;

class AgendaPesertaResource extends Resource
{
    protected static ?string $model = AgendaPeserta::class;

    protected static ?string $navigationLabel = 'Agenda Peserta';

    // Ikon menggunakan BackedEnum|string|null (sesuai parent)
    protected static string|BackedEnum|null $navigationIcon = Heroicon::OutlinedUserPlus;

    protected static ?string $recordTitleAttribute = 'label';

    // Soft delete aware query untuk Resource
    public static function getEloquentQuery(): Builder
    {
        return AgendaPeserta::query()->withoutGlobalScopes([
            SoftDeletingScope::class,
        ]);
    }

    public static function form(Schema $schema): Schema
    {
        return method_exists(AgendaPesertaForm::class, 'configure')
            ? AgendaPesertaForm::configure($schema)
            : AgendaPesertaForm::make($schema);
    }

    public static function table(Table $table): Table
    {
        return method_exists(AgendaPesertasTable::class, 'configure')
            ? AgendaPesertasTable::configure($table)
            : AgendaPesertasTable::make($table);
    }

    public static function getPages(): array
    {
        return [
            'index'  => ListAgendaPesertas::route('/'),
            'create' => CreateAgendaPeserta::route('/create'),
            'edit'   => EditAgendaPeserta::route('/{record}/edit'),
        ];
    }

    // Soft-delete aware binding untuk Edit
    public static function getRecordRouteBindingEloquentQuery(): Builder
    {
        return parent::getRecordRouteBindingEloquentQuery()
            ->withoutGlobalScopes([SoftDeletingScope::class]);
    }
}
```

### 4) Schema/Form

**app/Filament/Resources/AgendaPesertas/Schemas/AgendaPesertaForm.php**

```php
<?php

namespace App\Filament\Resources\AgendaPesertas\Schemas;

use App\Models\Agenda;
use App\Models\AgendaPeserta;
use App\Models\Participant;
use Filament\Forms;
use Filament\Schemas\Schema;

class AgendaPesertaForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema
            ->schema([
                Forms\Components\Select::make('participant_id')
                    ->label('E-Mail Peserta')
                    ->options(fn () => Participant::query()->orderBy('email')->pluck('email', 'id'))
                    ->searchable()
                    ->required(),

                Forms\Components\Select::make('agenda_id')
                    ->label('Agenda Pelatihan')
                    ->options(function () {
                        return Agenda::query()
                            ->get()
                            ->sortByDesc(fn ($a) => $a->event_date ?? $a->date ?? $a->start_time ?? $a->created_at)
                            ->mapWithKeys(function ($a) {
                                $date  = $a->event_date ?? $a->date ?? null;
                                $start = $a->hour_start ?? $a->start_time ?? null;
                                $end   = $a->hour_end ?? $a->end_time ?? null;
                                $title = $a->title ?? 'Agenda';
                                $label = $title
                                    . ($date ? " — {$date}" : '')
                                    . ($start ? " {$start}" : '')
                                    . ($end ? " - {$end}" : '');
                                return [$a->id => $label];
                            });
                    })
                    ->searchable()
                    ->required()
                    ->helperText('Tidak boleh ada kombinasi Email + Agenda yang sama.'),

                // Toggle hadir → set kolom attended_at (disimpan via hidden)
                Forms\Components\Toggle::make('hadir_now')
                    ->label('Tandai Hadir Sekarang')
                    ->helperText('ON = set Waktu Hadir ke sekarang, OFF = kosongkan.')
                    ->live()
                    ->dehydrated(false)
                    ->afterStateUpdated(function ($state, $set, $get) {
                        $set('attended_at', $state ? now('Asia/Jakarta') : null);
                    }),
                Forms\Components\Hidden::make('attended_at'),

                // Toggle rilis sertifikat → wajibkan nomor & tanggal
                Forms\Components\Toggle::make('rilis_now')
                    ->label('Rilis Sertifikat')
                    ->live()
                    ->dehydrated(false)
                    ->afterStateUpdated(function ($state, $set, $get) {
                        if ($state && ! $get('certificate_published_at')) {
                            $set('certificate_published_at', now('Asia/Jakarta'));
                        }
                    }),

                Forms\Components\TextInput::make('certificate_number')
                    ->label('Nomor Sertifikat')
                    ->required(fn ($get) => (bool) $get('rilis_now'))
                    ->maxLength(191),

                Forms\Components\DateTimePicker::make('certificate_published_at')
                    ->label('Sertifikat Terbit')
                    ->seconds(false)
                    ->native(false)
                    ->timezone('Asia/Jakarta')
                    ->displayFormat('d, F Y H:i')
                    ->required(fn ($get) => (bool) $get('rilis_now')),

                Forms\Components\Select::make('status')
                    ->label('Status')
                    ->options([1 => 'Aktif', 2 => 'Draft'])
                    ->default(1)
                    ->required(),

                Forms\Components\Textarea::make('description')
                    ->label('Deskripsi')
                    ->rows(3),
            ])
            ->columns(2)
            ->model(AgendaPeserta::class);
    }
}
```

### 5) Table

**app/Filament/Resources/AgendaPesertas/Tables/AgendaPesertasTable.php**

```php
<?php

namespace App\Filament\Resources\AgendaPesertas\Tables;

use App\Filament\Resources\AgendaPesertas\AgendaPesertaResource;
use App\Models\AgendaPeserta;
use Filament\Tables;
use Filament\Tables\Table;
use Filament\Tables\Filters\TrashedFilter;

// Actions: versi global "Filament\Actions\..."
use Filament\Actions\EditAction;
use Filament\Actions\BulkActionGroup;
use Filament\Actions\DeleteBulkAction;
use Filament\Actions\ForceDeleteBulkAction;
use Filament\Actions\RestoreBulkAction;

class AgendaPesertasTable
{
    public static function configure(Table $table): Table
    {
        return $table
            // Query eksplisit + eager load (hindari N+1)
            ->query(fn () => AgendaPeserta::query()->with(['participant', 'agenda']))
            ->columns([
                Tables\Columns\TextColumn::make('id')->label('ID')->sortable(),
                Tables\Columns\TextColumn::make('participant.email')->label('E-Mail')->sortable()->searchable(),
                Tables\Columns\TextColumn::make('agenda.title')->label('Agenda')->sortable()->searchable(),
                Tables\Columns\TextColumn::make('attended_at')->label('Waktu Hadir')->dateTime('d, F Y H:i', 'Asia/Jakarta')->sortable(),
                Tables\Columns\TextColumn::make('certificate_published_at')->label('Sertifikat')->dateTime('d, F Y H:i', 'Asia/Jakarta')->sortable(),
                Tables\Columns\TextColumn::make('certificate_number')->label('No. Sertifikat')->searchable(),
                Tables\Columns\TextColumn::make('created_at')->label('Dibuat')->dateTime('d, F Y H:i', 'Asia/Jakarta')->sortable(),
            ])
            ->filters([
                TrashedFilter::make(),
            ])
            ->recordActions([
                EditAction::make()->label('Ubah')
                    ->url(fn ($record) => AgendaPesertaResource::getUrl('edit', ['record' => $record])),
            ])
            ->bulkActions([
                BulkActionGroup::make([
                    DeleteBulkAction::make()->label('Hapus'),
                    ForceDeleteBulkAction::make()->label('Hapus Permanen'),
                    RestoreBulkAction::make()->label('Pulihkan'),
                ]),
            ]);
    }
}
```

### 6) Pages (Create & Edit) — validasi duplikat + error binding & toast

**app/Filament/Resources/AgendaPesertas/Pages/CreateAgendaPeserta.php**

```php
<?php

namespace App\Filament\Resources\AgendaPesertas\Pages;

use App\Filament\Resources\AgendaPesertas\AgendaPesertaResource;
use App\Models\AgendaPeserta;
use Filament\Notifications\Notification;
use Filament\Resources\Pages\CreateRecord;

class CreateAgendaPeserta extends CreateRecord
{
    protected static string $resource = AgendaPesertaResource::class;

    protected function beforeCreate(): void
    {
        $data = $this->form->getState();

        $exists = AgendaPeserta::query()
            ->where('participant_id', $data['participant_id'] ?? null)
            ->where('agenda_id', $data['agenda_id'] ?? null)
            ->whereNull('deleted_at')
            ->exists();

        if ($exists) {
            // tempel ke field select (schemas → pakai "data.*")
            $this->addError('data.agenda_id', 'Kombinasi Email + Agenda sudah ada.');

            Notification::make()
                ->title('Kombinasi Email + Agenda sudah ada.')
                ->danger()
                ->send();

            $this->halt();
        }
    }

    protected function mutateFormDataBeforeCreate(array $data): array
    {
        unset($data['rilis_now'], $data['hadir_now']);
        return $data;
    }

    protected function getCreatedNotificationTitle(): ?string
    {
        return 'Agenda Peserta berhasil dibuat';
    }
}
```

**app/Filament/Resources/AgendaPesertas/Pages/EditAgendaPeserta.php**

```php
<?php

namespace App\Filament\Resources\AgendaPesertas\Pages;

use App\Filament\Resources\AgendaPesertas\AgendaPesertaResource;
use App\Models\AgendaPeserta;
use Filament\Notifications\Notification;
use Filament\Resources\Pages\EditRecord;

class EditAgendaPeserta extends EditRecord
{
    protected static string $resource = AgendaPesertaResource::class;

    protected function beforeSave(): void
    {
        $data = $this->form->getState();

        $exists = AgendaPeserta::query()
            ->where('participant_id', $data['participant_id'] ?? null)
            ->where('agenda_id', $data['agenda_id'] ?? null)
            ->where('id', '<>', $this->record->id)
            ->whereNull('deleted_at')
            ->exists();

        if ($exists) {
            $this->addError('data.agenda_id', 'Kombinasi Email + Agenda sudah ada.');

            Notification::make()
                ->title('Kombinasi Email + Agenda sudah ada.')
                ->danger()
                ->send();

            $this->halt();
        }
    }

    protected function mutateFormDataBeforeSave(array $data): array
    {
        unset($data['rilis_now'], $data['hadir_now']);
        return $data;
    }

    protected function getSavedNotificationTitle(): ?string
    {
        return 'Agenda Peserta berhasil disimpan';
    }
}
```

> **Catatan**: Pastikan file Pages lainnya (`ListAgendaPesertas`) namespace-nya `App\Filament\Resources\AgendaPesertas\Pages`.

---

## Tests

* **Create** kombinasi baru → sukses.
* **Create** dengan kombinasi **email+agenda** sudah ada → **gagal** + toast merah + pesan merah di field Agenda.
* **Edit** ubah **Deskripsi** saja → sukses (tidak kena validasi unik).
* **Edit** ubah ke kombinasi yang sudah ada → **gagal** + toast merah + pesan merah di field Agenda.
* **Filter Trash** berfungsi jika ada record soft-deleted.

---

## Next Steps (Runbook)

1. Pastikan time zone:

   ```env
   APP_TIMEZONE=Asia/Jakarta
   ```
2. Jalankan perintah:

   ```bash
   php artisan migrate         # jika migration belum dijalankan
   php artisan storage:link    # jika butuh akses storage (opsional)
   php artisan optimize:clear
   ```
3. Buka panel admin: `/admin/agenda-pesertas`.

---

## N+1 & Kinerja

* Table menggunakan `->query(fn () => AgendaPeserta::query()->with(['participant','agenda']))` untuk eager loading.
* Untuk dropdown `Agenda`, data disortir di PHP lalu di-map agar tidak bergantung nama kolom spesifik yang mungkin berbeda.
* Jika data `agendas` besar, pertimbangkan:

  * Ubah `Select` menjadi `->relationship('agenda', 'title')` dengan `->searchable()` + `getSearchResultsUsing()` lazy (TODO jika dibutuhkan).
  * Tambahkan cache ringan untuk opsi agenda (valid di window pendek).

---

## Lokalization (ID)

* Label form & table sudah menggunakan Bahasa Indonesia: “E-Mail Peserta”, “Agenda Pelatihan”, “Waktu Hadir”, “Nomor Sertifikat”, “Sertifikat Terbit”, “Deskripsi”, dll.
* Teks notifikasi & error: “Kombinasi Email + Agenda sudah ada.”, “Agenda Peserta berhasil dibuat/disimpan”.

---

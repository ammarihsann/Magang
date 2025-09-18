* CRUD Agenda (Filament v4):

  * Kolom: `content_id`, `title`, `description`, `start_at`, `end_at`, `status` (active|draft), `created_by`, `updated_by`, `deleted_by`, timestamps, soft deletes.
  * Tabel: kolom `ID`, `Konten`, `Judul`, `Event Date` (tanggal saja), `Waktu Pelatihan` (HH\:mm–HH\:mm), `Dibuat`; klik baris ⇒ Edit.
  * Aksi baris: **Publish**, **Set Draft**, **Hapus** (soft delete), **Pulihkan** (hanya jika trashed).
  * Filter: Rentang tanggal, Status (active/draft), Trashed.
* Form:

  * Input tanggal & jam terpisah: `event_date` (DatePicker), `start_time` & `end_time` (TimePicker/TextInput time).
  * Saat simpan, compose ke `start_at` & `end_at` (simpan UTC; UI pakai Asia/Jakarta).
  * Validasi: `end_time` > `start_time`.
* Role/Policy:

  * **Super Admin** → full access.
  * **Editor** → boleh lihat semua; **hanya** bisa edit/hapus/restore agendanya sendiri.
* Kualitas:

  * Eager load `content` untuk cegah N+1.
  * Soft delete-aware; “Pulihkan” hanya muncul saat record trashed.
* Tests (Pest) disediakan untuk policy & compose waktu.

---

# Code

> Catatan: sesuaikan jika beberapa file sudah ada di projectmu. Path di bawah sudah final.

## /database/migrations/2025\_01\_01\_000000\_create\_agendas\_table.php

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('agendas', function (Blueprint $table) {
            $table->id();
            $table->foreignId('content_id')->constrained('contents')->cascadeOnUpdate()->restrictOnDelete();
            $table->string('title', 200);
            $table->text('description')->nullable();
            $table->dateTime('start_at')->index();
            $table->dateTime('end_at')->index();
            $table->string('status', 16)->default('draft')->index(); // active|draft
            $table->foreignId('created_by')->nullable()->constrained('users')->nullOnDelete();
            $table->foreignId('updated_by')->nullable()->constrained('users')->nullOnDelete();
            $table->foreignId('deleted_by')->nullable()->constrained('users')->nullOnDelete();
            $table->timestamps();
            $table->softDeletes();

            $table->index(['content_id', 'status']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('agendas');
    }
};
```

## /app/Models/Agenda.php

```php
<?php

namespace App\Models;

use App\Models\Traits\Blameable;
use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Agenda extends Model
{
    use HasFactory, SoftDeletes, Blameable;

    protected $fillable = [
        'content_id',
        'title',
        'description',
        'start_at',
        'end_at',
        'status',
        'created_by',
        'updated_by',
        'deleted_by',
    ];

    protected $casts = [
        'start_at' => 'datetime',
        'end_at'   => 'datetime',
    ];

    // normalize status
    protected function status(): Attribute
    {
        return Attribute::make(
            set: fn ($v) => $v === null ? null : strtolower(trim((string) $v)),
        );
    }

    public function content()
    {
        return $this->belongsTo(Content::class);
    }

    // Accessors untuk tampilan tabel
    public function getEventDateAttribute(): string
    {
        return optional($this->start_at)->timezone('Asia/Jakarta')->format('Y/m/d') ?? '-';
    }

    public function getEventTimeWindowAttribute(): string
    {
        $s = $this->start_at?->timezone('Asia/Jakarta');
        $e = $this->end_at?->timezone('Asia/Jakarta');
        if (! $s || ! $e) return '-';
        return $s->format('H:i') . ' - ' . $e->format('H:i');
    }
}
```

## /app/Policies/AgendaPolicy.php

```php
<?php

namespace App\Policies;

use App\Enums\AdminRole;
use App\Models\Agenda;
use App\Models\User;

class AgendaPolicy
{
    public function before(User $user, string $ability): ?bool
    {
        if ($user->role === AdminRole::SuperAdmin) {
            return true;
        }
        return null;
    }

    public function viewAny(User $user): bool
    {
        return in_array($user->role, [AdminRole::SuperAdmin, AdminRole::Editor], true);
    }

    public function view(User $user, Agenda $agenda): bool
    {
        return in_array($user->role, [AdminRole::SuperAdmin, AdminRole::Editor], true);
    }

    public function create(User $user): bool
    {
        return in_array($user->role, [AdminRole::SuperAdmin, AdminRole::Editor], true);
    }

    public function update(User $user, Agenda $agenda): bool
    {
        return $agenda->created_by === $user->id;
    }

    public function delete(User $user, Agenda $agenda): bool
    {
        return $agenda->created_by === $user->id;
    }

    public function restore(User $user, Agenda $agenda): bool
    {
        return $agenda->created_by === $user->id;
    }

    public function forceDelete(User $user, Agenda $agenda): bool
    {
        return false;
    }
}
```

## /app/Providers/AuthServiceProvider.php (potongan penting)

```php
<?php

namespace App\Providers;

use App\Models\Agenda;
use App\Policies\AgendaPolicy;
// ... other imports
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    protected $policies = [
        // ...
        Agenda::class => AgendaPolicy::class,
    ];

    public function boot(): void
    {
        //
    }
}
```

## /database/factories/AgendaFactory.php

```php
<?php

namespace Database\Factories;

use App\Models\Agenda;
use App\Models\Content;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class AgendaFactory extends Factory
{
    protected $model = Agenda::class;

    public function definition(): array
    {
        $startWib = now('Asia/Jakarta')->addDays(1)->setTime(9, 0);
        $endWib   = (clone $startWib)->setTime(11, 0);

        return [
            'content_id'  => Content::factory(),
            'title'       => $this->faker->sentence(3),
            'description' => $this->faker->sentence(),
            'start_at'    => $startWib->clone()->timezone('UTC'),
            'end_at'      => $endWib->clone()->timezone('UTC'),
            'status'      => 'draft',
            'created_by'  => User::factory(),
        ];
    }

    public function ownedBy(User $user): self
    {
        return $this->state(fn () => ['created_by' => $user->id]);
    }

    public function active(): self
    {
        return $this->state(fn () => ['status' => 'active']);
    }
}
```

## /app/Filament/Resources/Agendas/AgendaResource.php

```php
<?php

namespace App\Filament\Resources\Agendas;

use App\Filament\Resources\Agendas\Pages\CreateAgenda;
use App\Filament\Resources\Agendas\Pages\EditAgenda;
use App\Filament\Resources\Agendas\Pages\ListAgendas;
use App\Filament\Resources\Agendas\Schemas\AgendaForm;
use App\Filament\Resources\Agendas\Tables\AgendasTable;
use App\Models\Agenda;
use BackedEnum;
use Filament\Resources\Resource;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Table;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\SoftDeletingScope;

class AgendaResource extends Resource
{
    protected static ?string $model = Agenda::class;

    protected static string|BackedEnum|null $navigationIcon = Heroicon::OutlinedRectangleStack;

    protected static ?string $recordTitleAttribute = 'title';

    public static function form(Schema $schema): Schema
    {
        return AgendaForm::configure($schema);
    }

    public static function table(Table $table): Table
    {
        return AgendasTable::configure($table);
    }

    public static function getRelations(): array
    {
        return [];
    }

    public static function getPages(): array
    {
        return [
            'index'  => ListAgendas::route('/'),
            'create' => CreateAgenda::route('/create'),
            'edit'   => EditAgenda::route('/{record}/edit'),
        ];
    }

    public static function getRecordRouteBindingEloquentQuery(): Builder
    {
        return parent::getRecordRouteBindingEloquentQuery()
            ->withoutGlobalScopes([SoftDeletingScope::class]);
    }
}
```

## /app/Filament/Resources/Agendas/Schemas/AgendaForm.php

```php
<?php

namespace App\Filament\Resources\Agendas\Schemas;

use App\Models\Content;
use Filament\Forms\Components\DatePicker;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\TimePicker;
use Filament\Schemas\Schema;

class AgendaForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            Select::make('content_id')
                ->label('Konten Pelatihan')
                ->options(fn () => Content::query()->orderBy('title')->pluck('title', 'id')->toArray())
                ->searchable()
                ->preload()
                ->required(),

            TextInput::make('title')
                ->label('Judul Agenda')
                ->required()
                ->maxLength(200),

            Select::make('status')
                ->label('Status')
                ->options(['active' => 'Aktif', 'draft' => 'Draft'])
                ->required()
                ->rule('in:active,draft')
                ->afterStateHydrated(fn ($set, $record) => $set('status', $record?->status ?? 'draft'))
                ->dehydrated(true),

            Textarea::make('description')
                ->label('Deskripsi')
                ->rows(4)
                ->columnSpanFull(),

            DatePicker::make('event_date')
                ->label('Event Date')
                ->native(false)
                ->displayFormat('Y/m/d')
                ->required()
                ->afterStateHydrated(function ($set, $record) {
                    if ($record?->start_at) {
                        $set('event_date', $record->start_at->timezone('Asia/Jakarta')->format('Y-m-d'));
                    }
                }),

            TimePicker::make('start_time')
                ->label('Mulai (Jam)')
                ->seconds(false)
                ->native(false)
                ->required()
                ->afterStateHydrated(function ($set, $record) {
                    if ($record?->start_at) {
                        $set('start_time', $record->start_at->timezone('Asia/Jakarta')->format('H:i'));
                    }
                }),

            TimePicker::make('end_time')
                ->label('Selesai (Jam)')
                ->seconds(false)
                ->native(false)
                ->required()
                ->afterStateHydrated(function ($set, $record) {
                    if ($record?->end_at) {
                        $set('end_time', $record->end_at->timezone('Asia/Jakarta')->format('H:i'));
                    }
                }),
        ]);
    }
}
```

## /app/Filament/Resources/Agendas/Tables/AgendasTable.php

```php
<?php

namespace App\Filament\Resources\Agendas\Tables;

use App\Enums\AdminRole;
use App\Filament\Resources\Agendas\AgendaResource;
use App\Models\Agenda;
use Filament\Actions\Action;
use Filament\Actions\BulkActionGroup;
use Filament\Actions\DeleteAction;
use Filament\Actions\DeleteBulkAction;
use Filament\Actions\RestoreAction;
use Filament\Actions\RestoreBulkAction;
use Filament\Notifications\Notification;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\Filter;
use Filament\Tables\Filters\SelectFilter;
use Filament\Tables\Filters\TrashedFilter;
use Filament\Tables\Table;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\SoftDeletingScope;
use Illuminate\Support\Facades\DB;

class AgendasTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->query(fn (): Builder => Agenda::query()
                ->withoutGlobalScopes([SoftDeletingScope::class])
                ->with(['content'])
            )
            ->recordUrl(function ($record) {
                $user = auth()->user();
                return ($user && $user->can('update', $record))
                    ? AgendaResource::getUrl('edit', ['record' => $record])
                    : null;
            })
            ->columns([
                TextColumn::make('id')
                    ->label('ID')
                    ->sortable()
                    ->alignCenter()
                    ->width('80px'),

                TextColumn::make('content.title')
                    ->label('Konten')
                    ->sortable()
                    ->searchable(),

                TextColumn::make('title')
                    ->label('Judul')
                    ->limit(40)
                    ->sortable()
                    ->searchable(),

                TextColumn::make('event_date')
                    ->label('Event Date')
                    ->sortable(query: fn (Builder $q, string $dir) => $q->orderBy('start_at', $dir)),

                TextColumn::make('event_time_window')
                    ->label('Waktu Pelatihan'),

                TextColumn::make('created_at')
                    ->label('Dibuat')
                    ->dateTime('d, F Y H:i')
                    ->timezone('Asia/Jakarta')
                    ->sortable(),
            ])
            ->filters([
                Filter::make('rentang_tanggal')
                    ->label('Rentang Tanggal')
                    ->form([
                        \Filament\Forms\Components\DatePicker::make('from')->label('Dari'),
                        \Filament\Forms\Components\DatePicker::make('until')->label('Sampai'),
                    ])
                    ->query(function (Builder $query, array $data): Builder {
                        return $query
                            ->when($data['from'] ?? null, fn ($q, $date) => $q->whereDate('start_at', '>=', $date))
                            ->when($data['until'] ?? null, fn ($q, $date) => $q->whereDate('end_at', '<=', $date));
                    }),

                SelectFilter::make('status')
                    ->label('Status')
                    ->options(['active' => 'Aktif', 'draft' => 'Draft']),

                TrashedFilter::make()->label('Sampah'),
            ])
            ->defaultSort('start_at', 'desc')
            ->recordActions([
                Action::make('publish')
                    ->label('Publish')
                    ->icon('heroicon-o-check-badge')
                    ->color('success')
                    ->visible(fn ($record) => ! $record->trashed()
                        && auth()->user()?->can('update', $record)
                        && strtolower($record->status ?? '') !== 'active')
                    ->requiresConfirmation()
                    ->modalHeading('Publish Agenda')
                    ->modalDescription('Ubah status agenda menjadi Aktif?')
                    ->action(function (Agenda $record) {
                        DB::transaction(fn () => $record->update(['status' => 'active']));
                        Notification::make()->title('Agenda dipublish')->success()->send();
                    }),

                Action::make('set_draft')
                    ->label('Set Draft')
                    ->icon('heroicon-o-pencil')
                    ->color('warning')
                    ->visible(fn ($record) => ! $record->trashed()
                        && auth()->user()?->can('update', $record)
                        && strtolower($record->status ?? '') !== 'draft')
                    ->requiresConfirmation()
                    ->modalHeading('Set Draft')
                    ->modalDescription('Ubah status agenda menjadi Draft?')
                    ->action(function (Agenda $record) {
                        DB::transaction(fn () => $record->update(['status' => 'draft']));
                        Notification::make()->title('Agenda diubah ke Draft')->success()->send();
                    }),

                DeleteAction::make()
                    ->label('Hapus')
                    ->visible(fn ($record) => ! $record->trashed() && auth()->user()?->can('delete', $record)),

                RestoreAction::make()
                    ->label('Pulihkan')
                    ->visible(fn ($record) => $record->trashed() && auth()->user()?->can('restore', $record)),
            ])
            ->toolbarActions([
                BulkActionGroup::make([
                    DeleteBulkAction::make()->label('Hapus')
                        ->visible(fn () => auth()->user()?->role === AdminRole::SuperAdmin),
                    RestoreBulkAction::make()->label('Pulihkan')
                        ->visible(fn () => auth()->user()?->role === AdminRole::SuperAdmin),
                ]),
            ])
            ->emptyStateHeading('Belum ada Agenda')
            ->emptyStateDescription('Tambahkan agenda pelatihan pertama Anda.');
    }
}
```

## /app/Filament/Resources/Agendas/Pages/ListAgendas.php

```php
<?php

namespace App\Filament\Resources\Agendas\Pages;

use App\Filament\Resources\Agendas\AgendaResource;
use Filament\Actions\CreateAction;
use Filament\Resources\Pages\ListRecords;

class ListAgendas extends ListRecords
{
    protected static string $resource = AgendaResource::class;

    public function getTitle(): string
    {
        return 'Agenda Pelatihan';
    }

    protected function getHeaderActions(): array
    {
        return [
            CreateAction::make()->label('Buat Agenda'),
        ];
    }
}
```

## /app/Filament/Resources/Agendas/Pages/CreateAgenda.php

```php
<?php

namespace App\Filament\Resources\Agendas\Pages;

use App\Filament\Resources\Agendas\AgendaResource;
use Filament\Resources\Pages\CreateRecord;
use Illuminate\Support\Carbon;
use Illuminate\Validation\ValidationException;

class CreateAgenda extends CreateRecord
{
    protected static string $resource = AgendaResource::class;

    public function getTitle(): string
    {
        return 'Buat Agenda';
    }

    protected function getCreatedNotificationTitle(): ?string
    {
        return 'Agenda berhasil dibuat';
    }

    protected function getFormActions(): array
    {
        return [
            $this->getCreateFormAction()->label('Simpan'),
            $this->getCreateAnotherFormAction()->label('Simpan & Buat Lagi'),
            $this->getCancelFormAction()->label('Batal'),
        ];
    }

    protected function mutateFormDataBeforeCreate(array $data): array
    {
        $tz = 'Asia/Jakarta';

        $start = Carbon::parse($data['start_time'], $tz);
        $end   = Carbon::parse($data['end_time'], $tz);
        if ($end->lte($start)) {
            throw ValidationException::withMessages([
                'end_time' => 'Jam selesai harus lebih besar dari jam mulai.',
            ]);
        }

        $date = Carbon::parse($data['event_date'], $tz);
        $data['start_at'] = $date->copy()->setTimeFromTimeString($start->format('H:i'))->timezone('UTC');
        $data['end_at']   = $date->copy()->setTimeFromTimeString($end->format('H:i'))->timezone('UTC');

        $data['created_by'] = auth()->id();

        unset($data['event_date'], $data['start_time'], $data['end_time']);

        return $data;
    }
}
```

## /app/Filament/Resources/Agendas/Pages/EditAgenda.php

```php
<?php

namespace App\Filament\Resources\Agendas\Pages;

use App\Filament\Resources\Agendas\AgendaResource;
use Filament\Actions\DeleteAction;
use Filament\Actions\RestoreAction;
use Filament\Resources\Pages\EditRecord;
use Illuminate\Support\Carbon;
use Illuminate\Validation\ValidationException;

class EditAgenda extends EditRecord
{
    protected static string $resource = AgendaResource::class;

    public function getTitle(): string
    {
        return 'Ubah Agenda';
    }

    protected function getSavedNotificationTitle(): ?string
    {
        return 'Agenda berhasil diperbarui';
    }

    protected function getFormActions(): array
    {
        return [
            $this->getSaveFormAction()->label('Simpan'),
            $this->getCancelFormAction()->label('Batal'),
        ];
    }

    protected function getHeaderActions(): array
    {
        return [
            DeleteAction::make()->label('Hapus')->requiresConfirmation(),
            RestoreAction::make()->label('Pulihkan'),
        ];
    }

    protected function mutateFormDataBeforeSave(array $data): array
    {
        $tz = 'Asia/Jakarta';

        $start = Carbon::parse($data['start_time'], $tz);
        $end   = Carbon::parse($data['end_time'], $tz);
        if ($end->lte($start)) {
            throw ValidationException::withMessages([
                'end_time' => 'Jam selesai harus lebih besar dari jam mulai.',
            ]);
        }

        $date = Carbon::parse($data['event_date'], $tz);
        $data['start_at'] = $date->copy()->setTimeFromTimeString($start->format('H:i'))->timezone('UTC');
        $data['end_at']   = $date->copy()->setTimeFromTimeString($end->format('H:i'))->timezone('UTC');

        unset($data['event_date'], $data['start_time'], $data['end_time']);

        return $data;
    }
}
```

---

## (Optional) Tests – Pest

### /tests/Feature/AgendaPolicyTest.php

```php
<?php

use App\Enums\AdminRole;
use App\Models\Agenda;
use App\Models\User;

it('super admin can do everything', function () {
    $sa = User::factory()->create(['role' => AdminRole::SuperAdmin]);
    $agenda = Agenda::factory()->create();

    expect($sa->can('view', $agenda))->toBeTrue()
        ->and($sa->can('update', $agenda))->toBeTrue()
        ->and($sa->can('delete', $agenda))->toBeTrue()
        ->and($sa->can('restore', $agenda))->toBeTrue();
});

it('editor may view any but modify only own', function () {
    $editor = User::factory()->create(['role' => AdminRole::Editor]);
    $other  = User::factory()->create(['role' => AdminRole::Editor]);

    $own    = Agenda::factory()->ownedBy($editor)->create();
    $others = Agenda::factory()->ownedBy($other)->create();

    expect($editor->can('view', $others))->toBeTrue()
        ->and($editor->can('update', $own))->toBeTrue()
        ->and($editor->can('update', $others))->toBeFalse()
        ->and($editor->can('delete', $others))->toBeFalse()
        ->and($editor->can('restore', $others))->toBeFalse();
});
```

### /tests/Feature/AgendaCreateUpdateTest.php

```php
<?php

use App\Enums\AdminRole;
use App\Filament\Resources\Agendas\Pages\CreateAgenda;
use App\Filament\Resources\Agendas\Pages\EditAgenda;
use App\Models\Agenda;
use App\Models\Content;
use App\Models\User;
use Livewire\Livewire;

it('composes start_at & end_at from date and time (UTC stored)', function () {
    $user = User::factory()->create(['role' => AdminRole::SuperAdmin]);
    $content = Content::factory()->create();
    actingAs($user);

    Livewire::test(CreateAgenda::class)
        ->fillForm([
            'content_id' => $content->id,
            'title'      => 'Agenda Test',
            'description'=> 'Desc',
            'status'     => 'draft',
            'event_date' => '2025-09-18',
            'start_time' => '13:00',
            'end_time'   => '17:00',
        ])
        ->call('create')
        ->assertHasNoFormErrors();

    $a = Agenda::latest()->first();
    expect($a->start_at->timezone('UTC')->format('H:i'))->toBe('06:00')
        ->and($a->end_at->timezone('UTC')->format('H:i'))->toBe('10:00');
});

it('validates end time after start time on update', function () {
    $user = User::factory()->create(['role' => AdminRole::SuperAdmin]);
    $a = Agenda::factory()->ownedBy($user)->create();
    actingAs($user);

    Livewire::test(EditAgenda::class, ['record' => $a->getKey()])
        ->fillForm([
            'event_date' => '2025-09-18',
            'start_time' => '10:00',
            'end_time'   => '09:00',
        ])
        ->call('save')
        ->assertHasFormErrors(['end_time' => '']);
});
```

---

# Ops (Command & Runbook)

* (Jika belum ada) install dependencies: Filament v4 & Livewire 3 sudah ada di project kamu.
* Migrasi & seed:

  ```bash
  php artisan migrate
  php artisan db:seed   # jika punya seeder
  php artisan storage:link
  ```
* Role enum kamu: `App\Enums\AdminRole` dengan nilai `SuperAdmin|Editor` (sudah dipakai di policy).
* Timezone aplikasi: `.env` → `APP_TIMEZONE=Asia/Jakarta`.

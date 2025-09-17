# 📒 Laporan Harian — Manajemen Video & Konten Pelatihan (CMS)

Tanggal: **hari ini**
Stack: **Laravel 12 (PHP 8.4)**, **Filament v4 (Livewire 3)**, DB: **MySQL (InnoDB)**, TZ: **Asia/Jakarta**

---

## Ringkasan Pekerjaan

* Menyelesaikan modul **Manajemen Video & Konten** untuk admin (Filament Resource).
* Perbaikan bug **FileUpload spinner** (akar masalah: `.env` `APP_URL` salah).
* Aturan akses: **Editor** hanya bisa kelola konten miliknya; **Super Admin** bisa semua.
* **Restore per-record** lewat halaman **Edit** (untuk Editor).
  **Bulk actions** (delete/restore/force delete) hanya untuk **Super Admin**.
* Tabel: tampilkan status file **PDF/Video/HTML** sebagai ikon (✅/**—**) + tombol **preview** aman.
* Filter **Sampah (Trashed)** aktif; halaman **Edit** bisa membuka record yang sudah soft-deleted.

---

## Runbook & Konfigurasi

* `.env`

  ```env
  APP_URL=http://localhost.test
  FILESYSTEM_DISK=public
  ```
* Perintah:

  ```bash
  php artisan storage:link
  composer dump-autoload
  php artisan optimize:clear
  php artisan migrate
  ```

---

## Struktur Simpanan Berkas

```
storage/app/public/
└── contents/
    ├── thumbnails/YYYY/MM/
    ├── pdfs/YYYY/MM/
    ├── videos/YYYY/MM/
    └── html/YYYY/MM/
```

---

## Kode Lengkap

> Semua snippet di bawah ini **siap tempel** sesuai path.

### 1) Migration

**`database/migrations/2025_09_17_100157_create_contents_table.php`**

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('contents', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->text('description')->nullable();

            $table->string('thumbnail_url')->nullable();
            $table->string('content_pdf')->nullable();
            $table->string('original_pdf_filename')->nullable();
            $table->string('content_video')->nullable();
            $table->string('gamification_html')->nullable();

            // 1=Aktif, 2=Draft
            $table->tinyInteger('status')->default(2)->index();

            $table->foreignId('created_by')->constrained('users')->cascadeOnUpdate();
            $table->foreignId('updated_by')->nullable()->constrained('users')->cascadeOnUpdate();
            $table->foreignId('deleted_by')->nullable()->constrained('users')->cascadeOnUpdate();

            $table->timestamps();
            $table->softDeletes();

            $table->index(['created_by', 'deleted_at']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('contents');
    }
};
```

---

### 2) Model

**`app/Models/Content.php`**

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Support\Facades\Storage;

class Content extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'title',
        'description',
        'thumbnail_url',
        'content_pdf',
        'original_pdf_filename',
        'content_video',
        'gamification_html',
        'status',
        'created_by',
        'updated_by',
        'deleted_by',
    ];

    protected $casts = [
        'status' => 'integer',
    ];

    protected static function booted(): void
    {
        // set deleted_by saat soft delete
        static::deleting(function (self $content) {
            if (! $content->isForceDeleting()) {
                $content->timestamps = false;
                $content->deleted_by = auth()->id();
                $content->saveQuietly();
            }
        });

        // bersihkan file saat force delete
        static::forceDeleted(function (self $content) {
            $disk = Storage::disk('public');

            collect([
                $content->thumbnail_url,
                $content->content_pdf,
                $content->content_video,
                $content->gamification_html,
            ])->filter()->each(fn ($path) => $disk->delete($path));
        });
    }
}
```

---

### 3) Policy

**`app/Policies/ContentPolicy.php`**

```php
<?php

namespace App\Policies;

use App\Enums\AdminRole;
use App\Models\Content;
use App\Models\User;

class ContentPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, Content $content): bool
    {
        return true;
    }

    public function create(User $user): bool
    {
        return true;
    }

    public function update(User $user, Content $content): bool
    {
        return $user->role === AdminRole::SuperAdmin
            || $user->id === $content->created_by;
    }

    public function delete(User $user, Content $content): bool
    {
        return $user->role === AdminRole::SuperAdmin
            || $user->id === $content->created_by;
    }

    public function restore(User $user, Content $content): bool
    {
        if ($user->role === AdminRole::SuperAdmin) {
            return true;
        }

        return $user->id === $content->created_by;
    }

    public function forceDelete(User $user, Content $content): bool
    {
        return $user->role === AdminRole::SuperAdmin;
    }
}
```

**`app/Providers/AuthServiceProvider.php`**

```php
<?php

namespace App\Providers;

use App\Models\Content;
use App\Policies\ContentPolicy;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    protected $policies = [
        Content::class => ContentPolicy::class,
    ];

    public function boot(): void
    {
        //
    }
}
```

---

### 4) Filament Resource (v4)

**`app/Filament/Resources/Contents/ContentResource.php`**

```php
<?php

namespace App\Filament\Resources\Contents;

use App\Filament\Resources\Contents\Pages\CreateContent;
use App\Filament\Resources\Contents\Pages\EditContent;
use App\Filament\Resources\Contents\Pages\ListContents;
use App\Filament\Resources\Contents\Schemas\ContentForm;
use App\Filament\Resources\Contents\Tables\ContentsTable;
use App\Models\Content;
use Filament\Resources\Resource;
use Filament\Schemas\Schema;
use Filament\Tables\Table;

class ContentResource extends Resource
{
    protected static ?string $model = Content::class;

    protected static ?string $recordTitleAttribute = 'title';

    public static function getNavigationLabel(): string { return 'Contents'; }
    public static function getNavigationIcon(): ?string  { return 'heroicon-o-film'; }
    public static function getModelLabel(): string       { return 'Content'; }
    public static function getPluralModelLabel(): string { return 'Contents'; }

    public static function form(Schema $schema): Schema
    {
        return ContentForm::configure($schema);
    }

    public static function table(Table $table): Table
    {
        return ContentsTable::configure($table);
    }

    public static function getPages(): array
    {
        return [
            'index'  => ListContents::route('/'),
            'create' => CreateContent::route('/create'),
            'edit'   => EditContent::route('/{record}/edit'),
        ];
    }
}
```

#### 4.1 Form Schema

**`app/Filament/Resources/Contents/Schemas/ContentForm.php`**

```php
<?php

namespace App\Filament\Resources\Contents\Schemas;

use Filament\Schemas\Schema;
use Filament\Forms\Components\FileUpload;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\TextInput;

class ContentForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            TextInput::make('title')
                ->label('Judul')
                ->required()
                ->maxLength(255)
                ->columnSpanFull(),

            Textarea::make('description')
                ->label('Deskripsi')
                ->required()
                ->columnSpanFull(),

            FileUpload::make('thumbnail_url')
                ->label('Thumbnail')
                ->disk('public')
                ->visibility('public')
                ->image()
                ->maxSize(500) // KB
                ->rules([
                    'mimes:jpg,jpeg,png,gif',
                    'dimensions:max_width=1000,max_height=1000',
                ])
                ->directory(fn () => 'contents/thumbnails/' . now('Asia/Jakarta')->format('Y/m')),

            FileUpload::make('content_pdf')
                ->label('Konten PDF')
                ->disk('public')
                ->visibility('public')
                ->acceptedFileTypes(['application/pdf'])
                ->maxSize(10240) // 10 MB
                ->storeFileNamesIn('original_pdf_filename')
                ->directory(fn () => 'contents/pdfs/' . now('Asia/Jakarta')->format('Y/m')),

            FileUpload::make('content_video')
                ->label('Konten Video (MP4)')
                ->disk('public')
                ->visibility('public')
                ->acceptedFileTypes(['video/mp4'])
                ->maxSize(20480) // 20 MB
                ->directory(fn () => 'contents/videos/' . now('Asia/Jakarta')->format('Y/m')),

            FileUpload::make('gamification_html')
                ->label('Gamification HTML')
                ->disk('public')
                ->visibility('public')
                ->acceptedFileTypes(['text/html'])
                ->maxSize(1024) // 1 MB
                ->directory(fn () => 'contents/html/' . now('Asia/Jakarta')->format('Y/m')),

            Select::make('status')
                ->label('Status')
                ->options([1 => 'Aktif', 2 => 'Draft'])
                ->required()
                ->default(2),
        ]);
    }
}
```

#### 4.2 Table Schema

**`app/Filament/Resources/Contents/Tables/ContentsTable.php`**

```php
<?php

namespace App\Filament\Resources\Contents\Tables;

use App\Enums\AdminRole;
use App\Models\Content;
use Filament\Actions\BulkAction;
use Filament\Actions\BulkActionGroup;
use Filament\Notifications\Notification;
use Filament\Tables\Columns\IconColumn;
use Filament\Tables\Columns\ImageColumn;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\TrashedFilter;
use Filament\Tables\Table;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Gate;

class ContentsTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('id')->label('ID')->sortable(),

                ImageColumn::make('thumbnail_url')
                    ->label('Thumbnail')
                    ->disk('public')
                    ->height(40),

                TextColumn::make('title')
                    ->label('Judul')
                    ->searchable()
                    ->sortable()
                    ->url(fn (Content $record) => route('filament.admin.resources.contents.edit', ['record' => $record]))
                    ->openUrlInNewTab(false),

                IconColumn::make('has_pdf')
                    ->label('PDF')
                    ->state(fn (Content $r): bool => filled($r->content_pdf))
                    ->icon(fn (bool $state): string => $state ? 'heroicon-o-check-circle' : 'heroicon-o-minus')
                    ->color(fn (bool $state): string => $state ? 'success' : 'gray')
                    ->url(
                        fn (Content $r): ?string => $r->content_pdf ? route('admin.contents.file.show', [$r, 'pdf']) : null,
                        shouldOpenInNewTab: true
                    ),

                IconColumn::make('has_video')
                    ->label('Video')
                    ->state(fn (Content $r): bool => filled($r->content_video))
                    ->icon(fn (bool $state): string => $state ? 'heroicon-o-check-circle' : 'heroicon-o-minus')
                    ->color(fn (bool $state): string => $state ? 'warning' : 'gray')
                    ->url(
                        fn (Content $r): ?string => $r->content_video ? route('admin.contents.file.show', [$r, 'video']) : null,
                        shouldOpenInNewTab: true
                    ),

                IconColumn::make('has_html')
                    ->label('HTML')
                    ->state(fn (Content $r): bool => filled($r->gamification_html))
                    ->icon(fn (bool $state): string => $state ? 'heroicon-o-check-circle' : 'heroicon-o-minus')
                    ->color(fn (bool $state): string => $state ? 'info' : 'gray')
                    ->url(
                        fn (Content $r): ?string => $r->gamification_html ? route('admin.contents.file.show', [$r, 'html']) : null,
                        shouldOpenInNewTab: true
                    ),

                TextColumn::make('created_at')
                    ->label('Dibuat')
                    ->dateTime('d, F Y H:i', 'Asia/Jakarta')
                    ->sortable(),
            ])

            ->filters([
                TrashedFilter::make()->label('Sampah'),
            ])

            // cegah select jika tak berhak delete
            ->checkIfRecordIsSelectableUsing(fn (Content $record): bool => auth()->user()->can('delete', $record))

            ->bulkActions([
                BulkActionGroup::make([
                    // soft delete
                    BulkAction::make('hapusTerpilih')
                        ->label('Delete selected')
                        ->icon('heroicon-o-trash')
                        ->color('danger')
                        ->requiresConfirmation()
                        ->action(function (Collection $records): void {
                            $user = auth()->user();

                            $allowed = $records->filter(
                                fn (Content $r) => ! $r->trashed() && Gate::forUser($user)->allows('delete', $r)
                            );

                            $skipped = $records->count() - $allowed->count();

                            $allowed->each(fn (Content $r) => $r->delete());

                            if ($allowed->isNotEmpty()) {
                                Notification::make()->title("{$allowed->count()} konten dihapus")->success()->send();
                            }
                            if ($skipped > 0) {
                                Notification::make()->title("{$skipped} konten dilewati")->warning()->send();
                            }
                        }),

                    // restore
                    BulkAction::make('pulihkanTerpilih')
                        ->label('Restore selected')
                        ->icon('heroicon-o-arrow-uturn-left')
                        ->color('success')
                        ->requiresConfirmation()
                        ->action(function (Collection $records): void {
                            $user = auth()->user();

                            $allowed = $records->filter(
                                fn (Content $r) => $r->trashed() && Gate::forUser($user)->allows('restore', $r)
                            );

                            $skipped = $records->count() - $allowed->count();

                            $allowed->each(fn (Content $r) => $r->restore());

                            if ($allowed->isNotEmpty()) {
                                Notification::make()->title("{$allowed->count()} konten dipulihkan")->success()->send();
                            }
                            if ($skipped > 0) {
                                Notification::make()->title("{$skipped} konten dilewati (tidak berhak)")->warning()->send();
                            }
                        }),

                    // force delete
                    BulkAction::make('hapusPermanen')
                        ->label('Force delete selected')
                        ->icon('heroicon-o-trash')
                        ->color('danger')
                        ->requiresConfirmation()
                        ->action(function (Collection $records): void {
                            $allowed = $records->filter(fn (Content $r) => Gate::allows('forceDelete', $r));
                            $allowed->each(fn (Content $r) => $r->forceDelete());
                            Notification::make()
                                ->title($allowed->count() . ' konten dihapus permanen')
                                ->success()
                                ->send();
                        }),
                ])->visible(fn () => auth()->user()?->role === AdminRole::SuperAdmin),
            ]);
    }
}
```

#### 4.3 Pages

**`app/Filament/Resources/Contents/Pages/ListContents.php`**

```php
<?php

namespace App\Filament\Resources\Contents\Pages;

use App\Filament\Resources\Contents\ContentResource;
use Filament\Resources\Pages\ListRecords;

class ListContents extends ListRecords
{
    protected static string $resource = ContentResource::class;
}
```

**`app/Filament/Resources/Contents/Pages/CreateContent.php`**

```php
<?php

namespace App\Filament\Resources\Contents\Pages;

use App\Filament\Resources\Contents\ContentResource;
use Filament\Resources\Pages\CreateRecord;

class CreateContent extends CreateRecord
{
    protected static string $resource = ContentResource::class;

    protected function mutateFormDataBeforeCreate(array $data): array
    {
        $data['created_by'] = auth()->id();

        return $data;
    }
}
```

**`app/Filament/Resources/Contents/Pages/EditContent.php`**

```php
<?php

namespace App\Filament\Resources\Contents\Pages;

use App\Enums\AdminRole;
use App\Filament\Resources\Contents\ContentResource;
use Filament\Actions;
use Filament\Resources\Pages\EditRecord;
use Illuminate\Database\Eloquent\Model;

class EditContent extends EditRecord
{
    protected static string $resource = ContentResource::class;

    // Boleh buka record yang sudah soft-deleted supaya tombol Restore muncul
    protected function resolveRecord(int|string $key): Model
    {
        $record = static::getModel()::withTrashed()->findOrFail($key);
        $this->authorize('view', $record);

        return $record;
    }

    protected function mutateFormDataBeforeSave(array $data): array
    {
        $data['updated_by'] = auth()->id();
        return $data;
    }

    protected function getHeaderActions(): array
    {
        return [
            Actions\DeleteAction::make()
                ->visible(fn () => ! $this->getRecord()->trashed()
                    && auth()->user()->can('delete', $this->getRecord())),

            Actions\RestoreAction::make()
                ->label('Restore')
                ->visible(fn () => $this->getRecord()->trashed()
                    && auth()->user()->can('restore', $this->getRecord()))
                ->successNotificationTitle('Konten dipulihkan'),

            Actions\ForceDeleteAction::make()
                ->label('Hapus Permanen')
                ->color('danger')
                ->visible(fn () => $this->getRecord()->trashed()
                    && auth()->user()?->role === AdminRole::SuperAdmin)
                ->successNotificationTitle('Konten dihapus permanen'),
        ];
    }
}
```

---

### 5) Route & Controller untuk Preview/Download File

**`routes/web.php`** (tambahkan baris berikut)

```php
use App\Http\Controllers\Admin\ContentFileController;
use Illuminate\Support\Facades\Route;

Route::middleware(['web', 'auth'])->group(function () {
    Route::get('/admin/contents/{content}/{type}/file', [ContentFileController::class, 'show'])
        ->whereIn('type', ['pdf', 'video', 'html'])
        ->name('admin.contents.file.show');
});
```

**`app/Http/Controllers/Admin/ContentFileController.php`**

```php
<?php

namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use App\Models\Content;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\Facades\Storage;
use Symfony\Component\HttpFoundation\Response;

class ContentFileController extends Controller
{
    public function show(Request $request, Content $content, string $type)
    {
        Gate::authorize('view', $content);

        $map = [
            'pdf'   => $content->content_pdf,
            'video' => $content->content_video,
            'html'  => $content->gamification_html,
        ];

        $path = $map[$type] ?? null;

        if (! $path || ! Storage::disk('public')->exists($path)) {
            abort(Response::HTTP_NOT_FOUND);
        }

        $stream = Storage::disk('public')->readStream($path);
        $mime   = Storage::disk('public')->mimeType($path) ?? 'application/octet-stream';

        return response()->stream(function () use ($stream) {
            fpassthru($stream);
        }, 200, [
            'Content-Type'        => $mime,
            'Content-Disposition' => 'inline; filename="'.basename($path).'"',
            'Cache-Control'       => 'private, max-age=0, must-revalidate',
        ]);
    }
}
```

---

## Catatan Akhir / Backlog

* **Penghapusan file fisik saat soft delete**: **tidak dilakukan** (hanya saat **force delete** melalui event `forceDeleted()` di model). Ini mencegah kehilangan file jika user melakukan **restore**.
* Tambahkan observer/queue bila ukuran file besar dan perlu audit log.
* Integrasi **sertifikat** (PDF) menyusul: job antrean + tombol aksi di tabel.

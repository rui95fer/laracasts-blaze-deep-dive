# Blaze Deep-Dive — Laracasts Course Notes

---

## Episode 01 — Why Blaze Exists

- **Blade components have hidden overhead at scale — each one adds up.**
  ```blade
  {{-- A single page using small components everywhere --}}
  <x-button>Add to Cart</x-button>
  <x-icon name="chevron-right" />
  <x-badge :status="$user->role" />
  {{-- Innocent alone. 50 rows × 3 components = 150 renders per page load. --}}
  ```

- **The database is not always the bottleneck — Blade rendering often is.**
  ```bash
  # Before assuming slow queries, profile your view layer
  # Telescope, Debugbar, or even a simple benchmark can reveal Blade as the culprit
  php artisan telescope:install
  ```

- **Blaze is the fix — a drop-in package with performance strategies baked in.**
  ```bash
  # Install Blaze (covered in upcoming episodes)
  composer require livewire/blaze
  ```

---

## Episode 02 — Installing Blaze

- **Blade component loops inside other loops are the #1 performance killer.**
  ```blade
  {{-- A country select inside a loop of 250 users --}}
  @foreach ($users as $user)
      <x-select>
          @foreach ($countries as $country)   {{-- ~200 iterations --}}
              <x-option :value="$country" />  {{-- Blade component --}}
          @endforeach
      </x-select>
  @endforeach
  {{-- 250 users × 200 options = 50,000 Blade component renders --}}
  ```

- **A Blade component that does too much work (query, session, regex) becomes a time bomb when placed inside a loop.**
  ```blade
  {{-- Looks innocent, but fires a DB query 250 times --}}
  @foreach ($users as $user)
      <x-users-online :user="$user" />  {{-- queries last_login on every render --}}
  @endforeach
  ```

- **Install Blaze with a single Composer command — that's it.**
  ```bash
  composer require livewire/blaze
  ```

- **Always clear the compiled view cache after installing or changing anything Blade-related.**
  ```bash
  php artisan view:clear
  # Add this to your deploy script — it belongs there anyway
  ```

- **For Flux users: installing Blaze alone gives a ~25× speed improvement with zero other changes.**
  ```
  Before Blaze (Flux page, 250 users):  ~723ms
  After Blaze  (same page, no changes):  ~35ms
  ```

- **Blaze does not auto-optimize your own custom Blade components — you have to opt them in explicitly.**
  ```blade
  {{-- x-heading, x-text, x-badge etc. are NOT touched by Blaze out of the box --}}
  {{-- Next episode: how to tell Blaze to optimize your own components --}}
  ```

---

## Episode 03 — Three Levels of Optimization

- **Add `@blaze` to the top of a single component to opt it into Blaze's optimized compiler.**
  ```blade
  {{-- resources/views/components/option.blade.php --}}
  @blaze
  <option {{ $attributes }}>{{ $slot }}</option>
  ```
  ```bash
  php artisan view:clear
  ```

- **To optimize all components at once, use `Blaze::optimizeIn()` in your service provider instead of adding `@blaze` everywhere.**
  ```php
  // app/Providers/AppServiceProvider.php
  use Livewire\Blaze\Blaze;

  public function boot(): void
  {
      Blaze::optimizeIn(resource_path('views/components'));
  }
  ```

- **Blaze has three levels of performance — each one goes deeper.**
  ```
  Level 1 — Optimized compiler (default, just add @blaze)
      → ~2× speed improvement, removes most Blade overhead
      → 700ms → 314ms

  Level 2 — Memoization (memo: true)
      → Component renders once, result is reused everywhere else
      → Best for components with expensive work (queries, session lookups)
      → 314ms → 97ms

  Level 3 — Code folding (fold: true)
      → Goes even deeper (covered in later episodes)
      → 97ms → 52ms
  ```

- **Use `memo: true` on any component that does expensive work inside a loop — it renders once and caches the result.**
  ```blade
  {{-- resources/views/components/online-count.blade.php --}}
  @blaze(memo: true)
  {{-- This DB query now only runs once no matter how many times the component is used --}}
  <span>{{ DB::table('logins')->where(...)->count() }} online</span>
  ```

- **Use `fold: true` on static, repeated components like select options to collapse them at compile time.**
  ```blade
  {{-- resources/views/components/option.blade.php --}}
  @blaze(fold: true)
  <option {{ $attributes }}>{{ $slot }}</option>
  ```

- **Summary: Flux users get all three levels for free. Custom components start at Level 1 and you opt into deeper levels as needed.**
  ```
  Flux + Blaze installed:         ~35ms  (all levels automatic)
  Custom + @blaze on everything:  ~314ms  (Level 1 only)
  Custom + memo + fold:            ~52ms  (all three levels)
  ```

---

## Episode 04 — Benchmarking Blade

- **Blade component overhead is death by a thousand cuts — every layer adds cost.**
  ```
  Benchmark: rendering the same component 25,000 times

  Full Blade component (props, attributes, slot):  ~200ms
  Bare Blade component (no props, no merging):      ~80ms
  @include (bypasses component architecture):       ~44ms
  PHP require (bypasses Blade entirely):            ~12ms
  Inline HTML (no files, no Blade):                  ~3ms
  ```

- **The two biggest cost drivers inside a Blade component are attribute merging and `$slot`.**
  ```blade
  {{-- These two lines alone account for most of the overhead --}}
  @props(['type' => 'button'])

  <button {{ $attributes->merge(['class' => 'px-4 py-2']) }}>  {{-- most expensive --}}
      {{ $slot }}                                               {{-- second most expensive --}}
  </button>
  ```

- **`@include` is not just a PHP `require` — it has its own overhead on top.**
  ```blade
  {{-- Slower than you think — Blade does extra work around each include --}}
  @include('components.button')   {{-- ~44ms for 25k iterations --}}
  ```
  ```php
  // A raw PHP require is ~4× faster than @include
  require resource_path('views/components/button.blade.php');  // ~12ms
  ```

- **PHP itself is extremely fast at looping — the bottleneck is always Blade's layer, not PHP.**
  ```php
  // Pure PHP loop with inline HTML: ~3ms for 25,000 iterations
  // The moment you add any Blade layer on top, cost multiplies
  for ($i = 0; $i < 25000; $i++) {
      echo '<button>Click</button>';
  }
  ```

---

## Episode 05 — How The Blade Compiler Works

- **Blade is not PHP — it gets compiled into a raw PHP file before it runs.**
  ```blade
  {{-- Your source file: resources/views/benchmark.blade.php --}}
  {{ now() }}
  ```
  ```php
  // What PHP actually executes: storage/framework/views/abc123.php
  <?php echo e(now()); ?>
  ```

- **Compiled view files live in `storage/framework/views/` — you can inspect them anytime.**
  ```bash
  ls storage/framework/views/
  # Each file is a compiled PHP version of one of your Blade views
  ```

- **Blade recompiles a view by comparing file modified timestamps — not on every request.**
  ```
  resources/views/button.blade.php  ← last modified: 10:01am
  storage/framework/views/abc.php   ← last modified: 10:00am

  Source is newer → recompile.
  Source is same or older → use the cached compiled file.
  ```

- **`@include` compiles to `env()->make()` + `render()` — that's where its overhead comes from.**
  ```php
  // What @include('components.button') compiles to:
  echo $__env->make('components.button', array_diff_key(get_defined_vars(), ...))->render();
  // env()->make() resolves the view, array_diff_key extracts variables — neither is cheap
  ```

- **`<x-component>` compiles to even more steps — this is why it's 2× slower than `@include`.**
  ```php
  // What <x-button /> compiles to (simplified):
  $component = app()->make(\App\View\Components\Button::class);
  $component->withAttributes([...]);
  echo $component->resolveView()->render();
  // resolveAnonymousComponent, shouldRender, startComponent, renderComponent... lots of steps
  ```

- **Blaze's job is to skip as much of that compiled output as possible.**
  ```
  Vanilla Blade component → many PHP calls per render
  Blaze-optimized component → stripped-down compiled output, far fewer calls
  ```

---

## Episode 06 — Intercepting Blade Compilation

- **Blade exposes a `precompiler` hook that lets you intercept and mutate a view's raw string before it gets compiled.**
  ```php
  // app/Providers/AppServiceProvider.php
  use Illuminate\Support\Facades\Blade;

  public function boot(): void
  {
      Blade::precompiler(function (string $input): string {
          dd($input); // the entire raw string contents of the Blade file
      });
  }
  ```

- **Whatever you return from the precompiler is what Blade compiles — you fully control the output.**
  ```php
  Blade::precompiler(function (string $input): string {
      return 'foo'; // Blade will compile "foo" instead of the original file contents
  });
  ```

- **Use `preg_replace_callback()` to find and replace specific Blade tags inside the raw string.**
  ```php
  Blade::precompiler(function (string $input): string {
      return preg_replace_callback(
          '/<x-(?P<name>[\w-]+)(\s[^>]*)?\/>/',  // match self-closing Blade components
          fn ($matches) => "<!-- replaced: {$matches['name']} -->",
          $input
      );
  });
  ```

- **This is the exact mechanism Blaze uses under the hood — intercept the tag, compile it yourself, skip all the expensive Blade component machinery.**
  ```
  Normal flow:
    <x-button /> → Blade compiler → resolveAnonymousComponent, shouldRender, startComponent... → slow

  Blaze flow:
    <x-button /> → precompiler intercepts → custom compiled output → fast
  ```

---

## Episode 07 — Refactor Into A Compiler Class

- **Move precompiler logic out of the closure into a dedicated `Compiler` class to keep things clean.**
  ```php
  // app/Providers/AppServiceProvider.php
  Blade::precompiler(fn ($input) => (new Compiler)->compile($input));
  ```
  ```php
  class Compiler
  {
      protected string $pattern = '/<x-(?P<name>[\w-]+)(\s[^>]*)?\/?>/';

      function compile(string $input): string
      {
          return preg_replace_callback($this->pattern, $this->match(...), $input);
      }

      function match(array $groups): string
      {
          $name = $groups['name'];
          // ... build replacement output
      }
  }
  ```

- **Use PHP's first-class callable syntax (`$this->match(...)`) to pass a method as a clean callback.**
  ```php
  // Instead of a nested closure:
  preg_replace_callback($pattern, function ($groups) { ... }, $input);

  // Use first-class callable — cleaner, keeps nesting flat:
  preg_replace_callback($this->pattern, $this->match(...), $input);
  ```

- **Replace the matched tag with a raw PHP `require` pointing directly to the component file.**
  ```php
  function match(array $groups): string
  {
      $name = $groups['name'];
      $path = resource_path("views/components/{$name}.blade.php");

      return <<<PHP
      <?php require '{$path}'; ?>
      PHP;
  }
  ```

- **Requiring the raw `.blade.php` file only works if the component contains no Blade syntax — any `{{ }}` or `@` directives will break.**
  ```blade
  {{-- This component works with a raw require: --}}
  <button class="px-4 py-2">Click</button>

  {{-- This one does NOT — Blade syntax needs to be compiled first: --}}
  <button class="{{ $class }}">{{ $slot }}</button>
  ```

- **The next step is to compile the component file into PHP first, then require the compiled output — that's what the next episode covers.**
  ```
  Current:  require 'button.blade.php'     ← breaks with any Blade syntax
  Next:     require 'compiled/button.php'  ← works with everything
  ```

---

## Episode 08 — The Optimized Compiler

- **Never require a `.blade.php` file directly — compile it to PHP first, then require the compiled output.**
  ```php
  // Wrong: PHP can't understand Blade syntax
  require resource_path('views/components/button.blade.php');

  // Right: require the compiled PHP version
  require storage_path('framework/views/abc123.php');
  ```

- **Use `Blade::compileString()` to compile raw Blade contents into PHP without going through the filesystem.**
  ```php
  $source    = file_get_contents($sourcePath);
  $compiled  = Blade::compileString($source);
  ```

- **Hash the source path to generate a stable, unique filename for the compiled output — the same way Laravel does it.**
  ```php
  $hash         = substr(md5($sourcePath), 0, 8);
  $compiledPath = storage_path("framework/views/{$hash}.php");

  file_put_contents($compiledPath, $compiled);
  ```

- **Putting it all together: intercept the tag, compile the component, write it to storage, require it.**
  ```php
  function match(array $groups): string
  {
      $name         = $groups['name'];
      $sourcePath   = resource_path("views/components/{$name}.blade.php");

      $hash         = substr(md5($sourcePath), 0, 8);
      $compiledPath = storage_path("framework/views/{$hash}.php");

      $source   = file_get_contents($sourcePath);
      $compiled = Blade::compileString($source);

      file_put_contents($compiledPath, $compiled);

      return <<<PHP
      <?php require '{$compiledPath}'; ?>
      PHP;
  }
  ```

- **This is the core of what Blaze's optimized compiler does — skip all of Blade's component machinery and replace it with a plain `require` of a pre-compiled file.**
  ```
  Blade's default path:
    <x-button /> → resolveComponent → shouldRender → startComponent → render → slow

  Our toy compiler's path (and Blaze's real path):
    <x-button /> → compile source → write PHP file → require it → fast
  ```

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


---

## Episode 09 — The Blaze Directive

- **Check for `@blaze` in the component's source before optimizing — fall back to normal Blade if it's absent.**
  ```php
  function match(array $groups): string
  {
      $fallback   = $groups[0]; // the original tag, e.g. <x-button />
      $sourcePath = resource_path("views/components/{$groups['name']}.blade.php");

      if (! $this->shouldOptimize($sourcePath)) {
          return $fallback; // leave it for Blade to handle normally
      }

      return $this->compileComponent($sourcePath);
  }

  function shouldOptimize(string $path): bool
  {
      return str_contains(file_get_contents($path), '@blaze');
  }
  ```

- **Always return the original tag as the fallback — returning nothing removes the component from the output entirely.**
  ```php
  // Wrong: returns nothing, component disappears from the page
  if (! $this->shouldOptimize($path)) {
      return '';
  }

  // Right: returns the original tag so Blade handles it as normal
  if (! $this->shouldOptimize($path)) {
      return $groups[0];
  }
  ```

- **Strip `@blaze` from the source before compiling so it doesn't end up in the compiled PHP output.**
  ```php
  function compileComponent(string $sourcePath): string
  {
      $contents = file_get_contents($sourcePath);
      $contents = str_replace('@blaze', '', $contents); // remove the directive

      $hash         = substr(md5($sourcePath), 0, 8);
      $compiledPath = storage_path("framework/views/{$hash}.php");

      file_put_contents($compiledPath, Blade::compileString($contents));

      return "<?php require '{$compiledPath}'; ?>";
  }
  ```

- **With `@blaze` wired up, the directive becomes the hook for all deeper optimizations — `memo`, `fold`, and anything else.**
  ```blade
  @blaze                   {{-- Level 1: optimized compiler only --}}
  @blaze(memo: true)       {{-- Level 2: memoize this component --}}
  @blaze(fold: true)       {{-- Level 3: code fold this component --}}
  ```

---

## Episode 10 — How Fast Is Our Toy

- **The optimized compiler (Level 1) alone takes a full Blade component from ~87ms down to ~6ms for 25,000 renders.**
  ```
  Non-optimized (normal Blade):  ~87ms   for 25,000 iterations
  @blaze (optimized compiler):    ~6ms   for 25,000 iterations
  ```

- **Level 1 removes Blade's overhead — but any expensive work inside the component still runs on every render.**
  ```php
  // @blaze cuts Blade's machinery, but this usleep still runs 25,000 times
  @blaze
  <div>{{ $this->someExpensiveWork() }}</div>

  // 10 microseconds × 25,000 = 250ms of unavoidable overhead remaining
  ```

- **Common sources of per-render overhead that Level 1 cannot remove:**
  ```
  - Attribute bag merging
  - Large prop lists
  - match/switch statements
  - Auth checks (auth()->user())
  - Session lookups
  - Cache reads
  - Database queries
  ```

- **Memoization (Level 2) and code folding (Level 3) exist to eliminate that remaining per-render overhead — covered in the next episodes.**
  ```
  Level 1 — optimized compiler:  bypasses Blade machinery         → ~6ms
  Level 2 — memoization:         renders once, reuses the result  → eliminates repeated work
  Level 3 — code folding:        collapses static output at compile time → near-zero runtime cost
  ```

---

## Episode 11 — Adding Memoization

- **Memoization = in-memory caching for the duration of a single request — not a persistent cache.**
  ```php
  // Persistent cache (survives across requests):
  Cache::put('key', $value, now()->addHour());

  // Memoization (lives only for this request):
  Cache::rememberForever('key', fn () => $expensiveWork());
  // In Blaze's real implementation this is an in-memory store, not the cache driver
  ```

- **Use `Cache::rememberForever()` to wrap output — the closure only runs on the first render, every subsequent render returns the stored result.**
  ```php
  echo Cache::rememberForever('component.button', function () {
      ob_start();
      require $compiledPath;
      return ob_get_clean();
  });
  ```

- **Use output buffering (`ob_start` / `ob_get_clean`) to capture a component's rendered HTML into a variable so it can be stored and reused.**
  ```php
  ob_start();           // start capturing all output
  require $compiled;    // component renders into the buffer
  $html = ob_get_clean(); // grab the captured output as a string, stop buffering
  ```

- **The compiler wraps the `require` in memoization code when `@blaze(memo: true)` is detected.**
  ```php
  function compileComponent(string $sourcePath, string $name): string
  {
      // ... compile and write $compiledPath as before ...

      $key = "component.{$name}";

      return <<<PHP
      <?php echo cache()->rememberForever('{$key}', function () {
          ob_start();
          require '{$compiledPath}';
          return ob_get_clean();
      }); ?>
      PHP;
  }
  ```

- **Cache keys must include all component arguments to avoid collisions between different instances of the same component.**
  ```php
  // Wrong: two <x-badge status="active"> and <x-badge status="inactive">
  // share the same key and return identical output
  $key = "component.badge";

  // Right: hash the component name + all its attributes together
  $key = md5("badge" . serialize($attributes));
  ```

- **Output buffering is also how Blade's `$slot` works internally — slots capture echoed content into a variable and pass it into the component.**
  ```
  {{ $slot }} in a component
    → parent template echoes slot content into an output buffer
    → Blade captures it with ob_get_clean()
    → passes it as $slot variable into the child component
  ```

---

## Episode 12 — Code Folding

- **Code folding pre-renders a component at compile time so it costs nothing at runtime.**
  ```php
  // In the compiler's match() method:
  if ($this->shouldFold($sourcePath)) {
      ob_start();
      require $compiledPath; // render the component RIGHT NOW, at compile time
      return ob_get_clean(); // the rendered HTML becomes part of the compiled template
  }
  ```

- **The result: the component's HTML is baked directly into the parent template — no PHP runs at runtime at all.**
  ```
  Before folding (compiled output):
    <?php require '/storage/framework/views/abc.php'; ?>

  After folding (compiled output):
    <button class="px-4 py-2">Save</button>
    <!-- the component has vanished — it's just plain HTML now -->
  ```

- **Memoization vs folding — know when to use each.**
  ```
  Memoization — best for repeated, expensive components with dynamic output
    → renders once, caches the result, reuses it
    → still pays the cost once per request

  Code folding — best for static components with no dynamic data
    → renders once at compile time, costs zero at runtime
    → 37ms → 12ms → ~0ms
  ```

- **The concept comes from "constant folding" in early compilers — evaluate static expressions at compile time so the runtime never has to.**
  ```
  C compiler example:
    int x = 1 + 2;  →  compiled to:  int x = 3;
    (the addition never runs at runtime)

  Blaze equivalent:
    <x-icon name="trash" />  →  compiled to:  <svg>...</svg>
    (the component never renders at runtime)
  ```

- **Anything static in your templates is a candidate for folding — more than you'd expect.**
  ```blade
  {{-- These are ALL static — safe to fold: --}}
  <x-button>Save</x-button>
  <x-icon name="trash" />
  <x-badge status="active" />

  {{-- These are dynamic — cannot fold: --}}
  <x-button>{{ $label }}</x-button>
  <x-avatar :src="auth()->user()->avatar" />
  ```

---

## Episode 13 — From Toy To Real

- **Real Blaze uses the same `prepareStringsForCompilationUsing` hook as our toy — the entry point is identical.**
  ```php
  // Our toy:
  Blade::precompiler(fn ($input) => (new Compiler)->compile($input));

  // Real Blaze (simplified):
  Blade::prepareStringsForCompilationUsing(fn ($input) => Blaze::compile($input));
  ```

- **Real Blaze compiles each component into a PHP function, then calls it with props — our toy just uses a plain `require`.**
  ```php
  // Toy output:
  <?php require '/storage/framework/views/abc.php'; ?>

  // Real Blaze output (simplified):
  <?php Blaze::ensureRequired('/storage/framework/views/abc.php');
  blaze_button_abc(['class' => $class]); ?>
  // ensureRequired() skips the require if the file is already loaded — a small extra gain
  ```

- **Real Blaze's memoization is structurally identical to our toy — wraps the render in a key check, uses output buffering.**
  ```php
  // Toy:
  echo Cache::rememberForever('component.button', function () {
      ob_start(); require $compiled; return ob_get_clean();
  });

  // Real Blaze (simplified):
  $key = BlazeMemoizer::key('button', $props);
  if (BlazeMemoizer::has($key)) { echo BlazeMemoizer::get($key); }
  else { ob_start(); blaze_button_abc($props);
         BlazeMemoizer::put($key, ob_get_clean()); }
  ```

- **Real Blaze's folded output adds a source marker and cache-busting metadata — our toy just bakes the HTML in directly.**
  ```php
  // Toy fold output:
  <button class="px-4 py-2">Save</button>

  // Real Blaze fold output:
  <?php /* blaze-folded: button | source: ...button.blade.php | mtime: 1234567890 */ ?>
  <?php echo ltrim(ob_get_clean()); // ltrim fixes whitespace edge cases ?>
  <button class="px-4 py-2">Save</button>
  ```

- **Real Blaze parses templates into an AST instead of using a simple regex — but the compile → fold → memoize decision flow is the same.**
  ```
  Toy:   regex match → shouldOptimize → shouldFold → shouldMemoize → output
  Blaze: AST parse  → node fold      → memoize    → compile       → output
  // Same logic, production-grade parsing
  ```

---

## Episode 14 — Limitations And Edge Cases

- **Blaze does not support class-based components — only anonymous components.**
  ```bash
  # Class-based (NOT supported by Blaze):
  php artisan make:component Button
  # Creates app/View/Components/Button.php + resources/views/components/button.blade.php

  # Anonymous (supported — what everyone uses now):
  # Just create resources/views/components/button.blade.php with @props
  ```

- **`View::share()` variables are not available inside Blaze components — use `$__env->getShared()` as a workaround.**
  ```php
  // Service provider:
  View::share('brand', 'Hello Kitty');
  ```
  ```blade
  {{-- Breaks with @blaze: --}}
  {{ $brand }}

  {{-- Workaround: --}}
  {{ $__env->getShared()['brand'] }}
  ```

- **`@aware` only works when parent and child are both Blade or both Blaze — mixing breaks it.**
  ```blade
  {{-- Works: both are @blaze --}}
  <x-button-group size="lg">   {{-- @blaze --}}
      <x-button />             {{-- @blaze + @aware(['size']) --}}
  </x-button-group>

  {{-- Breaks: mixed Blade/Blaze boundary --}}
  <x-button-group size="lg">   {{-- plain Blade --}}
      <x-button />             {{-- @blaze + @aware(['size']) --}}
  </x-button-group>
  ```

- **`@aware` is an anti-pattern regardless of Blaze — avoid it because it inherits from any ancestor, not just the intended parent.**
  ```blade
  {{-- Intended: button-group passes size="lg" to buttons --}}
  <x-modal size="lg">               {{-- unrelated ancestor --}}
      <x-button-group>
          <x-button />              {{-- accidentally picks up size="lg" from modal --}}
      </x-button-group>
  </x-modal>
  {{-- @aware has no way to restrict which ancestor it listens to --}}
  ```

- **Full list of unsupported features — in practice none of these are commonly used.**
  ```
  ✗ Class-based components (php artisan make:component)
  ✗ View::share() — use $__env->getShared() workaround
  ✗ View composers on blade components
  ✗ Rendering blade components via view() helper
  ✗ @aware across Blade/Blaze boundaries
  ```

---

## Episode 15 — The Blade Profiler

- **Enable Blaze's built-in profiler via `.env` — works even with Blaze optimization disabled.**
  ```ini
  # .env
  BLAZE_ENABLED=true
  BLAZE_DEBUG=true
  ```
  ```bash
  php artisan view:clear  # always clear after changing Blaze settings
  ```

- **The profiler toolbar appears at the bottom of the page — click "Open Profiler" for the full flame chart.**
  ```
  Toolbar shows:
  - Total render time
  - Total component count
  - Slowest components ranked by total time
  ```

- **"Total time" includes all children; "Total self time" is the component alone — use self time to find the real culprit.**
  ```
  modal  →  total time: 750ms  |  total self time: 7ms
  ↑ modal itself is fast — its children (inputs, selects) are the problem
  ```

- **Profiler color codes tell you the optimization state of every component at a glance.**
  ```
  Blue   → unoptimized Blade component
  Orange → compiled by Blaze (Level 1)
  Green  → memoized (Level 2)   [shown in later episodes]
  Purple → folded (Level 3)     [shown in later episodes]
  Gray   → raw Blade view (not a component) — costs almost nothing
  ```

- **Enabling Blaze compilation alone (Level 1) shifts the bottleneck — use the profiler to find the new slowest component.**
  ```
  Before Blaze:
    Slowest: <x-option />     481ms  (Blade overhead dominates)
    Second:  <x-online-count> 200ms  (DB query)

  After Blaze Level 1:
    Slowest: <x-online-count> 200ms  (DB query — now exposed as the real culprit)
    Second:  <x-option />      40ms  (Blade overhead gone, only rendering cost left)
  ```

---

## Episode 16 — When To Memoize

- **Always reach for folding first — only fall back to memoization when folding isn't possible.**
  ```
  Decision flow:
    Can this component be frozen in time? → Yes → fold: true
    No → Is it repeated with stable props? → Yes → memo: true
    No → Level 1 optimized compiler only
  ```

- **Don't fold a component that reads from the database, session, auth, or URL — it will freeze the stale value permanently.**
  ```blade
  {{-- WRONG — folding freezes the query result in the compiled file --}}
  @blaze(fold: true)
  <span>{{ DB::table('user_sessions')->where(...)->count() }} online</span>

  {{-- RIGHT — memoize instead: fresh per request, free after the first render --}}
  @blaze(memo: true)
  <span>{{ DB::table('user_sessions')->where(...)->count() }} online</span>
  ```

- **Memoization caches for the duration of one request only — the value is always fresh on the next page load.**
  ```
  Request 1:  component renders → result cached in memory → reused for rest of request
  Request 2:  cache is empty → component renders fresh again
  ```

- **Memoization does not work with slots — skip it if the component uses `{{ $slot }}` or named slots.**
  ```blade
  {{-- memo is silently ignored / falls back when slots are present --}}
  @blaze(memo: true)
  <div class="card">
      {{ $slot }}  {{-- slots make memoization impossible --}}
  </div>
  ```

- **Memoization only helps when the same props are repeated — unique props per iteration create a new cache entry every time, making things slower.**
  ```blade
  {{-- No benefit — every user has a different ID, every render is a cache miss --}}
  @foreach ($users as $user)
      <x-online-count :user="$user" />
  @endforeach

  {{-- Great fit — same component, same props, rendered many times --}}
  @foreach ($posts as $post)
      <x-online-count />   {{-- no dynamic props, one cache entry reused everywhere --}}
  @endforeach
  ```

- **The best real-world use case for memoization is an avatar component — repeated everywhere, too dynamic to fold.**
  ```blade
  {{-- Flux's avatar: complex dynamic colors, used in every table row --}}
  {{-- Can't fold (dynamic), but memo saves it because it repeats per user --}}
  @blaze(memo: true)
  @props(['src', 'name'])
  <img src="{{ $src }}" class="rounded-full {{ $this->colorClass() }}" />
  ```

---

## Episode 17 — When To Fold

- **Always ask: "Is this a pure component?" before folding.**
  ```php
  // A pure component is like a pure function: it always returns the exact same output for the same inputs.
  function sum($a, $b) {
      return $a + $b; // Pure: Safe to fold
  }

  // It must not depend on external state, environment, or side effects.
  function sum($a, $b) {
      return $a + $b + DB::table('settings')->value('tax'); // Impure: Not safe to fold
  }
  ```

- **Common culprits that make a component impure and un-foldable:**
  ```blade
  {{-- These directives or helpers change based on global state, not component props --}}
  @error('email')             {{-- Error bags change per request / validation --}}
  @guest                      {{-- Auth state / guards change per user --}}
  {{ request()->url() }}      {{-- The URL / Route changes per request --}}
  {{-- Also: Session state, DB queries, Cache reads --}}
  ```

- **If a component is pure, mark it with `fold: true` to get massive performance gains.**
  ```blade
  {{-- resources/views/components/option.blade.php --}}
  {{-- Pure component: only depends on its inputs ($attributes, $slot) --}}
  @blaze(fold: true)
  <option {{ $attributes }}>{{ $slot }}</option>
  ```

- **Folding heavily repeated pure components collapses render graphs and drastically drops total time.**
  ```
  Example from the Profiler:
  An <x-option> component rendered 49,000 times inside large select fields.

  Without fold: ~77ms
  With fold:    ~15ms  (Overall page render time cut in half to 62ms)
  ```

- **Limitations ahead: just because you add `fold: true` doesn't mean Blaze is always able to fold it.**
  ```
  There are specific barriers and edge cases where Blaze will safely fall back to avoiding folding. (Covered in the next episode)
  ```

---

## Episode 18 — Folding With Dynamic Props

- **The Problem: Dynamic props break code folding.**
  ```blade
  {{-- Static props fold perfectly --}}
  <x-option value="US">USA</x-option>

  {{-- Dynamic props BREAK folding (Blaze doesn't know $code at compile time) --}}
  <x-option :value="$code">{{ $country }}</x-option>
  ```

- **The Solution: Partial Folding with `safe` props.**
  ```blade
  {{-- resources/views/components/option.blade.php --}}
  {{-- Tell Blaze that 'value' is a safe pass-through property --}}
  @blaze(fold: true, safe: ['value'])
  <option value="{{ $value }}">{{ $slot }}</option>
  ```

- **How partial folding works under the hood:**
  ```
  1. Blaze parses the blade file into an Abstract Syntax Tree (AST).
  2. It temporarily replaces your dynamic expression with a static string (e.g., `blaze_placeholder_0`).
  3. It pre-renders the component to static HTML using that placeholder.
  4. It uses string replacement to swap the dynamic expression back into the finalized HTML.
  ```

- **The Golden Rule: `safe` props must be strictly "pass-through".**
  ```
  They must enter the component and be directly echoed out (`{{ $value }}`). You cannot mutate, validate, or measure them inside the component, because at compile time, the component is interacting with the placeholder string, not the real runtime data.
  ```

- **Pitfalls & Practical Examples of breaking `safe` props:**
  ```blade
  {{-- 1. Conditionals fail --}}
  @if($size === 'sm')      {{-- Fails: It compares 'sm' to the string 'blaze_placeholder_0' --}}

  {{-- 2. String helpers return fake results --}}
  {{ strlen($value) }}     {{-- Returns ~20 (length of placeholder), not the length of "US" --}}

  {{-- 3. String mutation breaks the placeholder swap entirely --}}
  {{ strtolower($value) }} {{-- Output stays literal 'blaze_placeholder_0' because the case changed and Blaze's swap regex can't find it anymore --}}
  ```

---

## Episode 19 — Punching Holes With Unblaze

- **`fold: true` can over-freeze dynamic output; `@unblaze` is the escape hatch.**
  ```blade
  {{-- Folded component: everything becomes static at compile time --}}
  @blaze(fold: true)
  <div>{{ now()->format('s') }}</div> {{-- gets frozen --}}
  ```

- **Use `@unblaze ... @endunblaze` to punch a dynamic hole inside an otherwise folded component.**
  ```blade
  @blaze(fold: true)
  <div>
      @unblaze
          {{ now()->format('s') }} {{-- stays dynamic at runtime --}}
      @endunblaze
  </div>
  ```

- **Think of `@unblaze` as partial folding without `safe` placeholders: static shell, dynamic island.**
  ```
  Folded:   outer markup and static strings
  Dynamic:  only the code inside @unblaze
  Result:   keep most fold gains without freezing request-specific logic
  ```

- **Practical Livewire example: keep validation errors dynamic while folding input markup.**
  ```blade
  {{-- resources/views/components/input.blade.php --}}
  @blaze(fold: true)
  @props(['name', 'label'])

  <div>
      <label for="{{ $name }}">{{ $label }}</label>
      <input id="{{ $name }}" name="{{ $name }}" />

      @unblaze(scope: ['name' => $name])
          @error($scope['name'])
              <p class="text-red-600">{{ $message }}</p>
          @enderror
      @endunblaze
  </div>
  ```

- **Important caveat: `@unblaze` is a scope wall, so pass required variables through `scope:`.**
  ```blade
  {{-- Without scope: undefined variable errors are likely in the unblaze block --}}
  @unblaze(scope: ['name' => $name, 'url' => request()->path()])
      {{-- Access via $scope[...] inside this block --}}
  @endunblaze
  ```

- **Best real-world uses from the episode: error bags and current-URL checks in nav/input components.**
  ```
  - Validation messages: request-specific, must stay dynamic
  - Active nav states: URL-specific, must stay dynamic
  - Everything else: keep folded when possible
  ```

---

## Episode 20 — AI Powered Optimization

- **Use AI as an optimization assistant, not a replacement for judgment.**
  ```
  AI can automate most Blaze setup and tuning, but it may suggest unsafe folds/memoization.
  Human review is still required for request-specific behavior (auth, URL, errors, etc.).
  ```

- **Practical setup flow: install Boost, install skill, verify your IDE sees it.**
  ```bash
  composer require laravel/boost
  php artisan boost:install
  ```
  ```
  Then verify your AI client can see:
  - Laravel Boost MCP server
  - Blaze optimization skill
  ```

- **Fastest safe workflow: run trace-based optimization, not whole-codebase speculation.**
  ```
  1. Enable Blaze debug/profiler.
  2. Browse real slow pages in your app.
  3. Let AI optimize based on captured traces.
  4. Re-profile and compare before/after.

  Why: lower token cost, fewer blind guesses, better component-level decisions.
  ```

- **What AI usually gets right quickly:**
  ```
  - Enabling Blaze globally (e.g., optimize components directory)
  - Turning on debug mode for profiling
  - Suggesting memoization for repeated expensive components (like online counts)
  - Suggesting folding for highly repeated static components (like options/text wrappers)
  ```

- **What AI can miss (and what to manually verify):**
  ```
  - Choosing memo when fold is better
  - Missing `safe` pass-through props for dynamic attributes
  - Folding something request-specific and causing subtle UI regressions
  ```
  ```blade
  {{-- Example sanity check after AI edits --}}
  @blaze(fold: true, safe: ['value'])
  <option value="{{ $value }}">{{ $slot }}</option>
  ```

- **Practical review loop you can reuse on every page:**
  ```
  Prompt AI -> review each proposed edit -> accept/reject with intent
  -> run page again with profiler -> compare hot components
  -> patch remaining edge cases manually.
  ```
  ```
  Goal: get AI speed + human correctness.
  ```


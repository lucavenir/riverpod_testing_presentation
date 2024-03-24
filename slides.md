---
theme: the-unnamed
title: Scrivere Test con Riverpod e Mocktail
class: text-center
highlighter: shiki
drawings:
  persist: false
transition: fade
mdc: true
---

# Test con Riverpod e Mocktail

Manuale rapido per testare la tua logica

---
title: Chi sono?
layout: image-right
image: https://riverpod.dev/img/logo.png
backgroundSize: 70%
---

# Ciao!
<v-clicks>

**Chi sono?**

- 🎯 Dart enjoyer

- 🏞️ Riverpod enthusiast / supporter

- ⚗️ Functional programming lover

</v-clicks>

<v-clicks>

**Che si fa oggi?**

- 📚 Recap: cos'è Riverpod?

- 🐒 Cos'è Mocktail?

- 🧪 Scriviamo test

*bonus: Riverpod 3 spoilers!*

</v-clicks>


<div class="abs-br m-6 flex gap-2">
  <a href="https://github.com/lucavenir" target="_blank" alt="GitHub" title="~venir on GitHub"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
    <a href="https://twitter.com/venir_dev" target="_blank" alt="X" title="~venir on X"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-x />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)

You can have `style` tag in markdown to override the style for the current page.
Learn more: https://sli.dev/guide/syntax#embedded-styles
-->

---
---
# Che problema risolve Riverpod?


```dart {1|11|6|11|12-15|all}
class CheckPleaseWidget extends StatelessWidget {
  const CheckPleaseWidget({super.key});

  @override
  Widget build(BuildContext context) {
    final Future<int> bill = check();
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      mainAxisSize: MainAxisSize.min,
      children: [
        Text('Sono € ${2 * bill}, grazie 😸'),
        ElevatedButton(
          onPressed: moreSpritz,
          child: const Text('🍷'),
        ),
      ],
    );
  }
}
```

<!--
Un classico: attendiamo un risultato che in futuro, prima o poi (forse? e chi lo sa) arriverà.
Quanto vedete qui naturalmente NON va fatto, l'esempio è qui per stimolarci a volere:
- un modo per attendere il risultato e mostrare qualcosa all'utente nel frattempo
- un modo per isolare meglio la logica di business (non vogliamo il x2 nell'interfaccia!)
- un modo per gestire eventuali errori e mostrarli all'utente
- un modo per testare quanto sopra

Ah, se solo ci fosse un modo *davvero* semplice per isolare UI e Logica...
Ah, se solo potessimo iniettare, cachare e gestire dati in modo reattivo...
-->

---
---


---
---

# Business logic, with Riverpod

**Providers**: *functions with benefits*.
<br/>

<v-click>
Da ...
</v-click>
```dart {1|2|all}
Future<int> check() {
  return Future.delayed(200.milliseconds, () => ...);
}
```
<br/>
<v-click>
... a:
</v-click>

```dart {none|1|2|3|all}
@riverpod
Future<int> check(CheckRef ref) {
  return Future.delayed(200.milliseconds, () => ...);
}
```

---
---

# Business logic, with Riverpod
```dart {1-6|8-10|all}
@riverpod
class CheckController extends _$CheckController {
  @override
  Future<int> build() {
    return Future.delayed(200.milliseconds, () => 0);
  }

  Future<void> moreSpritz() {
    return update((state) => Future.delayed(200.milliseconds, () => state + 1));
  }
}
```

<br/>

```dart {none|1-2|3|4|all}
@riverpod
Future<int> fraudCheck(FraudCheckRef ref) async {
  final check = await ref.watch(checkProvider.future);
  return 2 * check;
}
```


---
---

# Final result

```dart {none|1-2|3|8-13|14-17|all}
class CheckPleaseWidget extends ConsumerWidget {
  Widget build(BuildContext context, WidgetRef ref) {
    final AsyncValue<int> fraud = ref.watch(fraudCheckProvider);
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      mainAxisSize: MainAxisSize.min,
      children: [
        switch (fraud) {
          AsyncData(value: 0) => const Text('Cosa ti porto? 👀'),
          AsyncData(:final value) => Text('Sono € $value, grazie 😸'),
          AsyncError() => const Text('Mi spiace, niente spriz oggi 😢'),
          _ => const Image.network('draft-wine-gif-url')
        },
        ElevatedButton(
          onPressed: ref.read(checkControllerProvider.notifier).moreSpritz,
          child: const Text('🍹'),
        ),
      ],
    );
  }
}
```

<div class="abs-tr m-6 flex gap-2">
  <a href="https://media.tenor.com/q_Pzz7xscmwAAAAM/glass-champagne-pouring-bubbley.gif" target="_blank" alt="Spritz" title="Draft a Spritz!"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    🍾
  </a>
</div>

---
---

# Testing setup

```dart {none|1-5|7-12|all}
ProviderContainer createContainer({List<Override> overrides = const []}) {
  final container = ProviderContainer(overrides: overrides);
  addTearDown(container.dispose);
  return container;
}

void main() {
  late ProviderContainer container;
  setUp(() {
    container = createContainer();
  });
}
```

<br/>

---
---

# Test suite

```dart {none|1|6|14|1-4|6-12|14-18|all}
  test("all'inizio il conto è zero", () async {
    final check = container.read(checkControllerProvider.future);
    await expectLater(check, completion(0));
  });

  test("posso caricare spritz sul conto del cliente", () async {
    var check = container.read(checkControllerProvider.future);
    await expectLater(check, completes);
    await container.read(checkControllerProvider.notifier).moreSpritz();
    var check = container.read(checkControllerProvider.future);
    await expectLater(check, completion(1));
  });

  test("posso truffare il mio cliente correttamente", () async {
    await container.read(checkControllerProvider.notifier).moreSpritz();
    final doubled = container.read(fraudCheckProvider);
    await expectLater(doubled, completion(2));
  });
```

<v-click>

⚠️ 2 test su 3 falliscono miseramente! ⚠️

</v-click>

---
---


# Mocktail
<v-click>

🍸

</v-click>

```dart {none|1-3|5|7-12|all}
abstract class Spritz {
  Spritz(this.amount);
  double amount;

  int max();

  bool drink(int sip) {
    assert(sip <= max(), "Well that's too much!");
    assert(sip <= amount, "No more left!");
    amount -= sip;
    return amount == 0;
  }
}
```
---
---

# Mocking

<v-clicks>

To Mock = "prendere in giro"

"fingere"
</v-clicks>

```dart {none|1-3|5|6|7|8|all}
import 'package:mocktail/mocktail.dart';

class Breezer extends Mock implements Spritz {}

void main() {
  final spritz = Breezer();
  final max = spritz.max();
  // error!
}
```

---
---

# Stubbing & Verifying

<v-clicks>

To Stub = "to extinguish a lighted cigarette by pressing the lighted end against something"

</v-clicks>


```dart {none|1-3|4|5-8|9-10|all}
void main() {
  test("I should be able to drink responsibly", () {
    final spritz = Breezer();
    when(() => spritz.max()).thenReturn(1);
    var done = spritz.drink(0.5);
    expect(done, false);
    done = spritz.drink(0.5);
    expect(done, true);
    verify(() => spritz.max()).called(2);
    verifyNoMoreInteractions(spritz);
  });
}
```

---
---

# Riverpod Recall

<v-clicks depth="2">

- 👀 `ref.watch`
  - "per *osservare* un provider"
  - keeps the provider alive
  - triggers a rebuild

- 👂 `ref.listen`
  - "per *ascoltare* un provider"
  - keeps the provider alive
  - no rebuild

- 📚 `ref.read`
  - per dare una letta un provider, senza effetti collaterali
</v-clicks>

---
---

# Riverpod 🤝 Mocktail

```dart {none|1-3|5-20|6|7|10|1-3|10|11-16|17|all}
class ProviderTransitionsTester<T> extends Mock {
  void call(T? previous, T value);
}

extension ListenerSetupExtension on ProviderContainer {
  ProviderTransitionsTester<T> testTransitionsOn<T>(
    ProviderListenable<T> provider, {
    bool fireImmediately = true,
  }) {
    final listener = ProviderTransitionsTester<T>();
    final subscription = listen(
      provider,
      listener.call,
      fireImmediately: fireImmediately,
    );
    addTearDown(subscription.close);
    return listener;
  }
}
```

---
---

# Initial state test

```dart {none|1|2-3|4|5|6|8|all|2|3|4-8|all}
test("all'inizio il conto è zero", () async {
  final tester = container.testTransitionsOn(checkControllerProvider);
  final _ = await container.read(checkControllerProvider.future);
  verifyInOrder([
    tester(null, const AsyncLoading()),
    tester(const AsyncLoading(), const AsyncData(0)),
  ]);
  verifyNoMoreInteractions(tester);
});
```

---
---

# Mutation test

```dart {none|1|2|3|4-9|all}
test("posso caricare spritz sul conto del cliente", () async {
  final tester = container.testTransitionsOn(checkControllerProvider);
  await container.read(checkControllerProvider.notifier).moreSpritz();
  verifyInOrder([
    tester(null, const AsyncLoading()),
    tester(const AsyncLoading(), const AsyncData(0)),
    tester(const AsyncData(0), const AsyncData(1)),
  ]);
  verifyNoMoreInteractions(tester);
});
```

---
---

# Reactive dependency test

```dart {none|1|2-4|13-16|5|6|7-11|all}
test("posso truffare il mio cliente correttamente", () async {
  final container = createContainer(
    overrides: [checkControllerProvider.overrideWith(MockedCheckController.new)],
  );
  final tester = container.testTransitionsOn(fraudCheckProvider);
  await container.read(fraudCheckProvider.future);
  verifyInOrder([
    tester(null, const AsyncLoading()),
    tester(const AsyncLoading(), const AsyncData(10)),
  ]);
  verifyNoMoreInteractions(tester);
});

class MockedCheckController extends CheckController {
  @override
  Future<int> check() async => 5;
}
```

---
---

# Wrap up

<v-clicks depth="2">

💭 Meglio scrivere qualche helper prima di iniziare

☣️ Alcune slide sono *volutamente* "imprecise"

🎯 It's a good starter project!

💬 Parliamone assieme!

- Riferimenti dalla documentazione:
  - [Unit tests](https://riverpod.dev/docs/essentials/testing#unit-tests)
  - [Spying changes on a provider](https://riverpod.dev/docs/essentials/testing#spying-on-changes-in-a-provider)
  - [Awaiting asynchronous providers](https://riverpod.dev/docs/essentials/testing#spying-on-changes-in-a-provider)
  - [Mocking Notifiers](https://riverpod.dev/docs/essentials/testing#mocking-notifiers)
- [Mocktail](https://pub.dev/packages/mocktail)

</v-clicks>

---
layout: center
---

# Grazie per l'attenzione


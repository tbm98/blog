---
title: "Quản lý trạng thái với Provider và StateNotifier"
description: "StateNotifier khá giống với ValueNotifier nhưng khắc phục được một số vấn đề của ValueNotifier."
date: 2020-06-13T22:45:57+07:00
draft: false
author: tbm98
tags:
 - flutter
---
Hello!
Nếu bạn đã từng sử dụng Provider thì chắc hẳn bạn cũng sẽ tận dụng luôn `ChangeNotifierProvider` hoặc `ValueListenableProvider` hoặc cái gì gì đó tương tự để quản lý "Nhà nước" luôn cho tiện đúng không.
`ChangeNotifier` thì mình đã dùng khá nhiều, đối với những screen có state đơn giản thì kiểm soát nó khá dễ và code cũng rất gọn, nhưng nếu với những screen có state phức tạp, cần quản lý rất nhiều trường thì nó dẫn tới tình trạng là toàn bộ `getters` vốn lẽ ra chỉ nên đặt tại `Model`/`State` thì giờ lại phải mang toàn bộ chúng paste vào `ChangeNotifier` thêm một lần nữa. Bên dưới là một đoạn code trong project cũ của mình.
```dart
class MainNotifier with ChangeNotifier {
  MainModel _mainModel;
  StreamController _streamController = StreamController();
  Notification _notification;

  MainNotifier() {
    _mainModel = MainModel();
    insertProfileGuest();
    loadCurrentMSV();
    loadAllProfile();
    _notification = Notification();
  }

  get getAllProfile => _mainModel.allProfile;

  get getClickedDay => DateTime(
      _mainModel.clickYear, _mainModel.clickMonth, _mainModel.clickDay);

  Stream get getStreamAction => _streamController.stream;

  Sink get inputAction => _streamController.sink;

  String get getTitle => _mainModel.title;

  get getDataSchedules => _mainModel.dataSchedules;

  get getWidth => _mainModel.width;

  get getItemWidth => _mainModel.itemWidth;

  get getItemHeight => _mainModel.itemHeight;

  get getDrawerHeaderHeight => _mainModel.drawerHeaderHeight;

  String get getName {
    if (_mainModel.msv == 'guest') {
      return 'Khách';
    }
    return _mainModel?.profile?.HoTen ?? 'Họ Tên';
  }

  String get getClass => _mainModel?.profile?.Lop ?? '';

  String get getToken => _mainModel?.profile?.Token ?? '';

  Map<String, List<Schedule>> get getEntriesOfDay => _mainModel.entriesOfDay;

  get getTableHeight => _mainModel.tableHeight;

  get getClickMonth => _mainModel.clickMonth;

  get getDateCurrentClick => _mainModel.getDateCurrentClick;

  get getKeyOfCurrentEntries => _mainModel.keyOfCurrentEntries;

  int get getCurrentDay => _mainModel.currentDay;

  int get getCurrentMonth => _mainModel.currentMonth;

  int get getCurrentYear => _mainModel.currentYear;

  int get getClickDay => _mainModel.clickDay;

  String get getMSV => _mainModel.msv;

  bool get isGuest =>
      getMSV == null ||
      getMSV == 'guest' ||
      getName == null ||
      getName == 'Họ Tên';

  String get getAvatarName {
    final name = getName;
    final splitName = name.split(' ');
    return splitName.last[0];
  }
```
Khá là dài dòng đúng không nhỉ :3
Đó là vấn đề của `ChangeNotifier`, mình nghĩ ra một cách giải quyết vấn đề đó là không expose toàn bộ các trường của state nữa mà thay vào đó thì chỉ expose duy nhất là state object thôi. Tới đây thì `ValueNotifier` sẽ là giải phát phù hợp hơn vì nó chỉ cần lắng nghe thay đổi của đúng một thằng là `State` thôi.
## Giới thiệu package
Mình mới đi lang thang trên mạng và tìm thấy thằng `StateNotifier` được giới thiệu là tương đồng với `ValueNotifier` nhưng khắc phục được một số vấn đề của `ValueNotifier` và `ChangeNotifier` nên tò mò sử dụng thử.
Một vài thông tin thêm về `StateNotifier` đó là tác giả của nó cũng chính là tác giả của package `Provider` nên mình cũng thêm phần tin tưởng :D
Sự khác nhau giữa `ValueNotifier` và `StateNotifier` bạn có thể xem tại đây [differences-with-valuenotifier](https://pub.dev/packages/state_notifier#differences-with-valuenotifier).
## Tạo một StateNotifier
Bây giờ hãy thử tạo một class `StateNotifier` của riêng mình để quản lý trạng thái nha.
`StateNotifier` là một abstract class nên mình phải tạo 1 class `extends StateNotifier<State>` trong đó State chính là `type` của state object của mình.
```dart
class MyStateNotifier extends StateNotifier<MyState> with LocatorMixin {

  // LocatorMixin là một mixin giúp class của mình giao tiếp với provider để lấy ra
  // được thứ mình cần mà không dùng tới context

  // Constructor mặc định nó yêu cầu gọi tới `super` và truyền vào một khởi tạo của state
  MyStateNotifier() : super(MyState(0, 1, 2, 3, name: 'tbm98'));

  // Ở đây chú ý: muốn StateNotifier nhận ra được có sự thay đổi của state
  // thì buộc chúng ta phải tạo ra một instance mới của state mỗi lần muốn cập
  // nhật một thứ gì đó.
  void increA() {
    state = state.copyWith(a: state.a + 1);
  }

  void increB() {
    state = state.copyWith(b: state.b + 1);
  }

  void increC() {
    state = state.copyWith(c: state.c + 1);
  }

  void increD() {
    state = state.copyWith(d: state.d + 1);
  }


  // @protected ở đây nghĩa là nó sẽ cảnh báo nếu như bạn
  // truy cập tới biến state từ một class không liên quan với lớp này
  // chi tiết hơn thì bạn có thể bật auto show documents when mouse move
  // và di chuyển con chuột vào từ đó hoặc giữ Ctrl/Cmd + click chuột phải
  // vào từ đó :D
  @override
  @protected
  set state(MyState value) {
    if (state != value) {
      // Ở đây có thể làm thứ gì đó mỗi khi state thay đổi
      // state là trạng thái cũ và value là trạng thái mới
      // như nói ở trên, nếu không tạo ra 1 instance khác thì
      // state sẽ luôn luôn = value và chả có gì xảy ra cả :D
      read<Logger>().countChanged(value.sumWithName);
    }
    super.state = value;
  }
}
```
Như mình đã comment trong code thì chúng ta cần chú ý rằng: muốn StateNotifier nhận ra được có sự thay đổi của state thì buộc chúng ta phải tạo ra một instance mới của state mỗi lần muốn cập nhật một thứ gì đó.
## Tạo một State
Vậy là mình đã tạo xong 1 lớp kết thừa `StateNotifier` rồi bây giờ sẽ tới bước tạo `State` chính là cái `class MyState` ở trên đó.
```dart
@freezed
abstract class MyState implements _$MyState {
  const MyState._();
  const factory MyState(int a, int b, int c, int d, {String name}) = _MyState;
  String get sumWithName {
    return '$name has ${a + b + c + d}';
  }
}
```
Như đã đề cập ở trên thì muốn thông báo rằng state thay đổi thì cần tạo ra một instance khác của state. Và mình lại tìm được 1 package nữa hỗ trợ tạo ra một object cứng (không có sự thay đổi về giá trị bên trong) tên là `Freezed` cũng do tác giả của `Provider` tạo ra luôn :D. Thông tin chi tiết về package này bạn có thể xem thêm tại đây [Freezed](https://pub.dev/packages/freezed).
Sở dĩ mình đưa phần tạo State này về sau là bởi ở `StateNotifier` cần tạo mới instance state mỗi lần update nên mình giới thiệu sau cho dễ hiểu.
Nếu như bạn không muốn sử dụng Freezed thì có thể tạo một class thế này
```dart
class MyState {
  const MyState(int a, int b, int c, int d, String name);
  final int a;
  final int b;
  final int c;
  final int d;
  final String name;
  String get sumWithName {
    return '$name has ${a + b + c + d}';
  }
}
```
## Tạo một View
Phần trạng thái và quản lý trạng thái đã xong rồi, bây giờ tới phần cuối dùng là phần `View`.
```dart
void main() {
  runApp(
    MultiProvider(
      providers: [
        Provider<Logger>(create: (_) => ConsoleLogger()),
        StateNotifierProvider<MyStateNotifier, MyState>(
          create: (_) => MyStateNotifier(),
          builder: (context, child) {
            return child;
          },
        ),
      ],
      child: MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return const MaterialApp(
      title: 'Flutter Demo',
      home: MyHomePage(),
    );
  }
}

class MyHomePage extends StatelessWidget {
  const MyHomePage({Key key}) : super(key: key);

  Widget _titleContent() {
    print('rebuild title content');
    return const Text(
      'You have pushed the button this many times:',
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Counter example'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            _titleContent(),
            Selector<MyState, int>(
              selector: (_, myState) => myState.a,
              builder: (_, value, __) {
                return Text(value.toString());
              },
            ),
            Divider(),
            Selector<MyState, int>(
              selector: (_, myState) => myState.b,
              builder: (_, value, __) {
                return Text(value.toString());
              },
            ),
            Divider(),
            Selector<MyState, int>(
              selector: (_, myState) => myState.c,
              builder: (_, value, __) {
                return Text(value.toString());
              },
            ),
            Divider(),
            Selector<MyState, int>(
              selector: (_, myState) => myState.d,
              builder: (_, value, __) {
                return Text(value.toString());
              },
            ),
          ],
        ),
      ),
      floatingActionButton: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          FloatingActionButton(
            onPressed: () {
              context.read<MyStateNotifier>().increA();
            },
            child: Text('A'),
          ),
          FloatingActionButton(
            onPressed: () {
              context.read<MyStateNotifier>().increB();
            },
            child: Text('B'),
          ),
          FloatingActionButton(
            onPressed: () {
              context.read<MyStateNotifier>().increC();
            },
            child: Text('C'),
          ),
          FloatingActionButton(
            onPressed: () {
              context.read<MyStateNotifier>().increD();
            },
            child: Text('D'),
          ),
        ],
      ),
    );
  }
}
```
Ở hàm main hình có setup `MultiProvider`, bạn để ý phần này nhé
```dart
        StateNotifierProvider<MyStateNotifier, MyState>(
          create: (_) => MyStateNotifier(),
          builder: (context, child) {
            return child;
          },
        )
```
Cái này cũng tương tự như `ChangeNotifierProvider` nhưng khác ở chỗ là nó cung cấp cùng lúc cả `StateNotifier` và `State`.
Có thể trong tương lai thì không cần khai báo hàm builder khi đặt trong MultiProvider nữa :3
Ở phần `View` thì vẫn sử dụng `Selector`/`Consumer`/`context.watch` ... như bình thường để có thể lấy ra được `StateNotifier`/`State` và sử dụng bình thường.
## Kết luận:
* Sử dụng `StateNotifier` sẽ làm cho phần quản lý trạng thái chỉ có chức năng là xử lý logic và cập nhật trạng thái tương ứng.
* State sẽ lưu trữ getters và các hàm liên quan chứ không đặt ở phần quản lý trạng thái nữa
* Có hiệu năng tốt hơn `ValueNotifier` (theo tác giả của package)
* Phân tách code giữa `View`/`Logic`/`Model` rõ ràng hơn `ChangeNotifier`

Toàn bộ source code, các thư viện liên quan bạn có thể xem tại [Đây](https://github.com/tbm98/flutter_packages_test/tree/master/state_notifier_test)
Hãy để lại ý kiến của bạn dưới phần bình luận nhé :D
---
name: react-native-best-practices
description: Chinese company baseline for performant React Native and Expo applications, including lists, animation, navigation, UI, state, and native dependencies.
license: MIT
metadata:
  based-on: react-native-best-practices
---

# React Native / Expo Skills

## 优先级

1. List performance
2. Animation
3. Navigation
4. UI patterns
5. State management
6. Rendering
7. Monorepo
8. Configuration

## 列表与动画

- 大列表优先评估 FlashList 或项目已有的 virtualized list。
- list item 使用稳定 callback、稳定 object reference 和必要的 memo。
- 图片、昂贵计算和 item type 需要根据实际列表性能处理。
- Reanimated 动画优先使用 transform、opacity 等 GPU-friendly properties。
- 派生动画值使用 `useDerivedValue`。
- 手势场景优先使用 `Gesture.Tap` 等 Gesture API，而不是堆叠 `Pressable` 逻辑。

## Navigation 与 UI

- 优先使用 native stack 和 native tabs，除非业务确实需要 JS navigator。
- 图片默认评估 `expo-image`；gallery 使用项目认可的 image lightbox。
- 优先使用 `Pressable`，不要继续新增 `TouchableOpacity`。
- ScrollView 正确处理 safe area 和 `contentInset`。
- 菜单、modal、view measurement 优先使用 native API、`onLayout` 和平台能力。
- 样式优先使用 `StyleSheet.create` 或项目统一的 Nativewind 方案。
- 文本必须位于 `Text` 组件内。
- 不要用 falsy `&&` 产生意外渲染。

## State 与工程结构

- 最小化 state subscription，区分 source of truth 与派生值。
- callback 复杂时使用 dispatcher pattern。
- 首次 render 提供稳定 fallback。
- 让 React Compiler 和 Reanimated shared values 的使用符合项目版本。
- native dependency 放在 app package，monorepo 使用统一版本。
- custom fonts 使用 config plugin；design system import 保持统一路径。

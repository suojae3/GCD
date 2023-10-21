---

![[Pasted image 20231011220439.png|500]]
- <mark style="background: #FF5582A6;">첫번째 반드시 메인큐에서 처리해야할 작업은 메인큐에서 처리해야한다</mark>.
- UI관련 일들은 메인큐에서 처리해야한다.
- <mark style="background: #FFF3A3A6;">화면처리는 반드시 한 개의 스레드에서만 담당해야지만 (직렬로 처리해야지만) 서로 간섭이 안일어남(간섭이 일어난 경우 화면이 깜빡일 수 있음) - 모든 OS 공통</mark>

<br/>

![[Pasted image 20231011220139.png|500]]
- 예를 들어 이미지를 다운해서 화면에 뿌려야하는 경우 
- 이미지다운은 비동기로 처리하되
- 이미지 보여주는 작업은 메인큐로 보내야한다.

<br/>


![[Pasted image 20231011220240.png|500]]
- URLSession의 경우 자동적으로 비동기처리되기 때문에 이 작업 안에서의 UI작업은 메인큐로 보내주어야한다.

---
---


```swift
var imageCache = [String: UIImage]()

class CustomImageView: UIImageView {

	var lastImgUsedToLoadImage: String?

	func loadImage(with urlString: String) {
		self.image = nil

		//마지막으로 이미지를 다운로드한 String 경로
		lastImgUrlUsedToLoadImage = urlString

		//이미지가 캐시에 들어있는지 확인하기
		if let cachedImage = imageCache[urlString] {
			self.image = cachedImage
			return
		}

		//url
		guard let url = URL(string: urlString) else { return }

		//URL세션은 내부적으로 비동기처리된 함수임
		URLSession.shared.dataTask(wit: url) { (data, response, error) in

			if let error = error {
				print("Failed to load image with error")
			}

			if self.lastImgUrlUsedToLoadImage != url.absoluteString {
				return
			}
	
			guard let imgData = data else { return }

			let photoImage = UIImage(data: imageData)

			imageCache[url.absoluteString] = photoImage

			//UI작업은 메인큐에서 ^^
			DispatchQueue.main.async {
				self.image = photoImage
			}
		}.resume()
	}
}
```

---
---


![[Pasted image 20231011223332.png|300]]
![[Pasted image 20231011223457.png|500]]
- Sync메서드 구현에 있어서 절대해서는 안되는 코드 <mark style="background: #FF5582A6;">두 가지</mark>가 존재
- <mark style="background: #FFB86CA6;">첫째, 메인큐에서 다른큐로 보낼때는 `sync`메서드를 부르면 절대안된다!</mark> (`async`메서드 사용)
- 다시말해 메인큐는 항상 비동기적으로 보내야한다.
- <mark style="background: #FFF3A3A6;">UI와 관련되지 않으면서 오래걸리는 작업 (ex: 네트워크 작업)들은 다른쓰레드에서 작업을 맡게 비동기적으로 처리를 해주어야하며</mark> 이를 동기적(sync)으로 처리할경우 UI버벅임 현상을 경험한다.

<br/>


![[Pasted image 20231011223856.png|400]]
![[Pasted image 20231011223821.png|500]]

- <mark style="background: #FFB86CA6;">두번째, 현재 큐 -> 현재큐로 동기적으로 보내서는 안된다!</mark>
- <mark style="background: #FFF3A3A6;">현재의 큐를 블락하는 동시에 다시 현재의 큐에 접근하면 교착상황(데드락)이 생긴다!</mark>

<br/>

![[Pasted image 20231011224116.png|500]]

- 대부분 개발자가 하는 실수는 위의 실수보다는 위와 같은 실수가 발생한다.
- 객체 안에 어떤 객체를 생성했는데 내부적으로 처리가 디폴트큐 sync처리되어있는 실수

---
---

![[Pasted image 20231011224357.png|500]]
- 다만 작업이 재할당될 때 2번이 아닌 3,4,5번에 할당되면 교착상태는 발생하지 않음
- 하지만 현재의 큐에서 현재의 큐를 동기적으로 보내면 안되는 이유는 교착상태의 가능성이 있기때문

--- 
---

```swift
//Person.swift
import Foundation


//Person 클래스
open class Person {
	private var firstName: String
	private var lastName: String

	public init(firstName: String, lastName: String) {
		self.firstName = firstName
		self.lastName = lastName
	}

	open func changeName(firstName: String, lastName: String) {

		DispatchQueue.global().sync {
			randomDelay(maxDuration: 0.2)
			self.firstName = firstName
			randomDelay(maxDuration: 1)
			self.lastName = lastName
		}
	}

	// 이름 Get메서드
	open var name: String {
		return DispatchQueue.global().sync {
			"\(firstName) \(lastName)"
		}
	}
}
```

---
---

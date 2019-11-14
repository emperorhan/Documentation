EOSIO IBC 원리와 디자인
---------
*저자:Simon [Github](https://github.com/vonhenry)*

이 문서는 boscore 팀이 개발한 IBC(contracts 및 ibc_plugin)의 기술 원칙을 소개합니다.

서로 다른 EOSIO 블록체인간에 token inter-blockchain transfer를 실현하려면 먼저 두 가지 문제를 해결해야합니다: 
1. Lightweight Client(lwc)를 실현하는 방법
2. double spend 및 replay attack을 방지하는 블록체인간 트랜잭션의 무결성과 신뢰성을 보장하는 방법

EOS 메인넷과 BOS 메인넷은 아래와 같습니다.
이 문서는 EOS 메인넷과 BOS 메인넷 두 개의 EOSIO 블록체인 아키텍처에 적용됩니다.


### 1. 전문용어
- BOSIBC  
  BOS Core 기술 팀에서 개발한 IBC contracts 및 ibc_plugin.
  
### 2. 핵심개념 및 자료구조
- Simple Payment Verification (SPV)  
  SPV는 거래가 블록체인에 존재하는지 확인하기 위해 Satoshi의 Bitcoin 백서에서 처음 제안되었습니다(https://bitcoin.org/bitcoin.pdf).
  `SPV client`는 인접한 블록헤더를 저장하지만 블록바디는 저장하지 않으므로 적은 양의 저장공간을 차지합니다.
  트랜잭션과 트랜잭션의 `merkle path`를 수신하면,
  `SPV client`는 트랜잭션의 머클패스를 사용하여 해당 트랜잭션이 블록체인에 존재하는지 여부를 확인할 수 있습니다.

- Lightwight client (lwc)  
  `SPV client'라고도하는 블록헤더로 구성된 경량 체인입니다.

- Merkle path  
  트랜잭션이 블록체인에 존재하는지 효율적으로 확인하기 위해서는 전체 블록 대신 트랜잭션의 원본 데이터와 트랜잭션 `merkle path` 만을 필요로합니다.(`merkle path`는 `merkle branch`라고도 불림)
  계산한 merkle root와 블록헤더의 merkle root가 동일하다면 트랜잭션이 해당 블록체인에 존재함을 나타냅니다.

- Block Producer Schedule  
  `BP Schedule` 은 EOSIO 블록체인 아키텍쳐 기술로 블록 생산자가 블록을 생성 할 수 있는 권한을 결정하는데 사용됩니다.
  새로운 버전의 `BP Schedule`은 마지막 `BP Schedule`의 유효성 검사를 통화 한 후에 적용됩니다.
  BP 권한에 대해 엄격한 핸드오버를 보장하기 위해 해당 Lightweight Client는 반드시 메인넷과 일치하는 `BP Schedule`을 따라야 합니다. 이는 IBC 시스템 로직에 핵심 기술입니다.

- forkdb
  EOSIO 노드에는 블록 정보를 저장하는데 사용되는 두 가지 기본 DB가 있습니다.
  하나는 `blog` 즉, 비가역 블록을 저장하는데 사용되는 `block log`이고 다른 하나는 가역 블록을 저장하는데 사용되는 `forkdb`입니다.
  `forkdb`는 블록정보의 일부를 현재 블록체인의 맨 위에 저장합니다.
  블록은 최종적으로 비가역 블록에 들어가기전에 반드시 `forkdb`에 의해 승인되어야합니다.
  Lightweight Client는 주로 `forkdb`의 로직를 나타냅니다.
  
### 3. Lightweight Client

크로스 체인 문제를 해결하기 위해 가장 먼저 해결되어야 할 문제는 Lightweight Client를 구현하는 방법입니다.
1. Lightweight Client는 어디에서 실행되어야 할까요? 스마트 컨트랙트에서? 아니면 플러그인과 같은 스마트 컨트랙트 외부에서?
2. 만약 Lightweight Client가 스마트 컨트랙트에서 실행되는 경우 상대 블록체인의 모든 블록헤더 데이터를 실시간으로 동기화하거나 필요에 따라 트랜잭션을 검증하기 위해 블록헤더 데이터의 일부를 동기화 해야합니다. 모든 블록헤더 데이터를 동기화하는 것은 연결된 각각의 체인의 상당히 많은 CPU 자원을 소비하기 때문에 필요에 따라 트랜잭션을 검증하기 위해 블록헤더 데이터의 일부를 동기화 해야합니다.
3. 그리고 relay node의 신뢰없이 Lightweight Client의 신뢰성을 보장해야하며 악의적인 공격을 방지해야하며 IBC의 완전한 탈 중앙화를 달성해야합니다.


#### 3.1 스마트 컨트랙트에서 Lightweight Client를 실행해야하는 경우
Bitcoin의 Lightweight Client는 원래 단일 노드(개인용 컴퓨터 또는 Decentralized Bitcoin mobile wallet)에서 실행되어 거래가 있는지 확인하고 해당 블록의 깊이를 확인했습니다.
이에 대한 구현 스펙은 [breadwallet](https://github.com/breadwallet/breadwallet-core)을 참조 할 수 있습니다.

IBC와 Decentralized Wallet에는 라이트 클라이언트에 대한 요구 사항이 다릅니다.
Decentralized Wallet은 일반적으로 개인 모바일 앱에서 실행되므로 개별 사용자에게 트랜잭션 확인 서비스를 제공하는 반면 IBC 시스템은 개방적이고 누구나 접근 가능하며 신뢰할 수 있는 라이트 클라이언트가 필요합니다.
이 관점에서 볼 때, 스마트 컨트랙트 데이터만이 글로벌하게 일관되며 조작이 불가능 하기 때문에 공개 신뢰를 얻을 수 있는 라이트 클라이언트는 스마트 컨트랙트에서만 실행할 수 있습니다.
스마트 컨트랙트 외부에서 신뢰할 수 있는 라이트 클라이언트를 구현할 수 없으므로 BOSIBC는 스마트 컨트랙트에서 라이트 클라이언트를 실행합니다.
[ibc.chain](https://github.com/boscore/ibc_contracts/tree/master/ibc.chain)의 소스 코드를 참조하십시오.


#### 3.2 모든 블록헤더를 동기화할지 여부
Bitcoin 및 ETH의 라이트 클라이언트에는 시작 블록과 일부 후속 검증 블록이 제공되며 라이트 클라이언트는 해당 시작 블록 이후의 전체 블록헤더를 동기화합니다.
Bitcoin은 연간 4Mb의 블록헤더만 생성합니다.
모바일 디바이스의 현재 저장 용량에 따르면, 연간 4Mb의 블록헤더 크기는 완전히 수용 될 수 있고, 이들 블록의 동기화는 모바일 디바이스의 많은 컴퓨팅 자원을 소비하지 않습니다.
그러나 EOSIO의 상황은 상당히 다릅니다.
EOSIO 스마트 컨트랙트는 스마트 컨트랙트에 블록헤더를 추가하려면 0.5ms의 CPU를 소비하고 블록헤더를 삭제하려면 0.2ms를 소비하므로 모든 블록헤더 처리에 0.7ms의 CPU가 필요합니다.
각 블록의 총 CPU 시간에 따라 다른 EOSIO 체인의 모든 블록헤더를 동기화한다고 가정하면 `0.7ms/200ms = 0.35%`로 전체 체인의 0.35%의 CPU 사용량을 사용해야합니다.(EOSIO 체인은 한 블록에 대해서 최대 200ms의 CPU를 사용할 수 있다)
전체 네트워크의 위임된 실제 총 토큰 총량은 4억개이므로 IBC 시스템이 작동하도록하려면 블록헤더 및 ibc 트랜잭션을 푸시하는 계정은 CPU에 `400 million * 0.35% = 1.4 million`(140만개)의 토큰을 위임해야 합니다.
EOSIO는 멀티 사이드 체인 생태계를 옹호하기 때문에 앞으로 크로스 체인을 달성하기 위해 여러 사이드 체인과 EOSIO 메인 네트워크가 있고 각 사이드 체인과 일대 다 크로스를 실현한다고 가정합시다.
메인넷과 10개의 사이드체인 모델을 가정 할 때, 각 체인은 10개의 라이트 클라이언트를 유지해야합니다. 이러한 라이트 클라이언트를 유지하려면 메인넷의 전체 네트워크 CPU의 3.5%를 소비해야합니다.
따라서 더 합리적인 솔루션이 필요합니다.

블록체인 간 커뮤니케이션을 설계하는 과정은 확실한 증거를 찾는 과정입니다.
전체 블록 정보를 동기화 할 필요는 없지만 라이트 클라이언트의 신뢰성을 보장 할 수 있는 체계가 있을까요?
EOSIO 소스 코드의 맨 아래에 이 목적을 위해 준비된 소스가 있습니다.
BP schedule이 변경되지 않는 경우, 언제라도 ibc.chain 스마트 컨트랙트가 서명 인증에 의해 전달된 일련의 블록헤더(예: `n ~ n+336`)를 포함하고 3분의 2 이상의 활성화된 BP가 블록을 생성하면 `n` 번째 블록이 비가역화 되었음을 확신할 수 있고 이를 크로스 체인 트랜잭션 검증에 사용할 수 있습니다.
이제 BP schedule이 변경 되었을 경우를 고려해야합니다.
BP schedule이 변경되면, 변경이 완료 될 때까지 트랜잭션 검증을 인정하지 않습니다.
비교적 복잡한 BP 교체 프로세스는 아래에서 더 자세히 설명 될 것입니다.
요점은 이와 같은 체계를 사용하면 동기화해야하는 블록헤더의 양을 크게 줄일 수 있으며 BP 목록이 업데이트되거나 크로스 체인 트랜잭션이 발생 되었을 때만 블록헤더가 필요하게 됩니다.

이 목표를 달성하기 위해서 `section` 개념이 ibc.chain에 도입되었습니다.
section은 일련의 연속된 블록헤더를 기록합니다.
section은 블록헤더를 저장하지 않는 대신에 블록헤더의 첫 번째 블록 번호(first)와 마지막 블록 번호(last)를 저장하며, 블록헤더는 `chaindb`에 저장됩니다.
각 section에는 유효한 boolean 값이 있습니다.
BP schedule 교체가 없는 경우 활성화된 BP의 3분의 2 이상이 블록을 생성하고 `last - first > lib_depth`이면 `first to last - lib_depth` 블록은 비가역화 된 것으로 간주되어 크로스 체인 트랜잭션을 검증하는데 사용 될 수 있습니다.
BP schedule이 교체되면 해당 section의 '유효성'이 거짓이 되고 이 section에 인정된 트랜잭션 검증은 BP schedule이 완료될 때까지 무효화 된다. BP schedule이 완료되면 section의 '유효성'이 참이 되고 크로스 체인 트랜잭션 검증을 진행할 수 있다.


#### 3.3 라이트 클라이언트의 안정성을 보장하는 방법
**3.3.1 forkdb**  
*1. forkdb에 새 블록을 추가하는 방법*  
실행중인 eosio 노드는 [blog](https://github.com/EOSIO/eos/blob/master/libraries/chain/include/eosio/chain/block_log.hpp)와 [forkdb](https://github.com/EOSIO/eos/blob/master/libraries/chain/include/eosio/chain/fork_database.hpp)라는 두 가지 기본 데이터 구조를 유지 관리하며 blog는 비가역된 블록 정보를 저장하는데 사용되고, 저장된 데이터는 serialize 된 `signed_block`이고, forkdb는 되돌릴 수 있는 블록에 대한 정보를 저장하는데 사용되며 저장된 데이터는 `block_state`입니다.
`block_state`는 `signed_block`보다 더 많은 블록 관련 정보를 포함합니다.
블록은 비가역화 되기 이전에 반드시 forkdb에 추가되어져야 합니다. 그리고 비가역화 될 때 forkdb에서 제거되고 blog에 포함됩니다.
블록의 `block_state` 정보는 어떻게 얻을 수 있을까요?
프로덕션 블록의 BP가 `block_state`를 다른 BP 노드 & Full 노드에게 전송하는 것은 아니며, P2P 네트워크인 EOSIO가 [signed_block](https://github.com/EOSIO/eos/blob/master/plugins/net_plugin/include/eosio/net_plugin/protocol.hpp#L142) 만을 전달합니다.
노드가 P2P로 `signed_block`을 전달 받으면 이 `signed_block`을 사용하여 `block_state`를 구성한 다음 BP 서명을 확인합니다.
[Relevance function](https://github.com/EOSIO/eos/blob/master/libraries/chain/block_header_state.cpp#L144)에 대하여 핵심 요점은 다음과 같습니다.
1. `blockroot_merkle`
2. `get_scheduled_producer()`
3. `verify_signee()`

서명 확인 관련 기능은 다음과 같습니다:
```
  digest_type   block_header_state::sig_digest()const {
     auto header_bmroot = digest_type::hash( std::make_pair( header.digest(), blockroot_merkle.get_root() ) );
     return digest_type::hash( std::make_pair(header_bmroot, pending_schedule_hash) );
  }

  public_key_type block_header_state::signee()const {
    return fc::crypto::public_key( header.producer_signature, sig_digest(), true );
  }

  void block_header_state::verify_signee( const public_key_type& signee )const {
     EOS_ASSERT( block_signing_key == signee, wrong_signing_key, "block not signed by expected key",
                 ("block_signing_key", block_signing_key)( "signee", signee ) );
  }
```

서명을 확인하는 첫 번째 단계는 서명 요약을 얻는 것입니다. (예: `header.digest()`, `blockroot_merkle.get_root()` 및 `pending_schedule_hash`를 사용하는 `sig_digest()`)
두 번째 단계는 서명 공개 키를 가져오는 것 입니다. 즉, `signee()`를 가져와서 `producer_signature ` 및 `sig_digest()`를 통해 BP 공개 키를 계산합니다.
세 번째 단계는 퍼블릭 키가 올바른지 확인하는 것 입니다. (예: `block_header_state:: next()`에서 호출되는 `verify_signee()`)
유효성 검사 후 블록은 forkdb main branch에 추가됩니다.

따라서 forkdb에 추가된 모든 블록은 매우 엄격하고 포괄적인 검증을 거쳤습니다.
핵심 논리에는 `blockroot_merkle`, `get_scheduled_producer()` 및 `verify_signee()`가 포함됩니다.
ibc.chain 스마트 컨트랙트는 forkdb의 엄격한 유효성을 완전히 상속합니다.

*2. forkdb가 포크를 처리하는 방법* 
새 블록을 추가하게 되면 forkdb의 head.id와 controller_impl의 head.id가 달라져 branch가 다시 선택됩니다.
소스 코드는 `eosio::chain::controller_impl::push_block()` 및 `eosio::chain::controller_impl::maybe_switch_forks()`를 나타냅니다. 

*3. LIB를 결정하는 방법*  
현재 EOSIO가 사용하는 합의 과정은 pipelined bft 라고도하는 dpos 입니다.
dpos는 `block_header_state` 블록을 구성 할 때 활성화된 BP의 2 / 3 + 1을 나타내는 `required_confs`를 설정합니다. (`required_confs`는 BP가 21명인 경우 15 입니다)
각 블록에는 이전 블록을 확인하는데 사용되는 `header.confirmed`가 있습니다.
블록이 컨펌을 받으면 `required_confs`는 1씩 줄어듭니다.
블록의 `required_confs`가 0으로 줄어들면 블록은 최신블록(forkdb의 head)에 의해 `dpos_proposed_irreversible_blocknum`으로 지정됩니다.
블록이 2/3 BP 지명을 받으면 비가역화 즉, LIB가 됩니다.
컨펌 정보는 블록헤더에 전달되므로 블록이 블록생성에서 LIB가 되는데 까지 총 2 * 2 / 3 라운드를 필요로하며 이는 `12 * (14 * 2) = 336` 개의 블록을 의미합니다.
BP가 등록되면 각 라운드마다 12개의 블록이 매번 지속적으로 생성됩니다.
첫 번째 블록의 `header.confirmed`만 0이 아니므로 BP가 블록에서 나오기 시작하면 첫 번째 블록만 LIB를 증가시킵니다.
따라서 실제 head와 LIB 간격은 325와 336 사이입니다.
그러나 BP가 블록을 손실할 경우 head와 LIB 간격은 325 내지 336 보다 작을 수 있습니다.

*4. BP 리스트 변경 방법*
Bitcoin 및 Ethereum과 같은 PoW 블록체인에서는 누적 컴퓨팅 성능이 가장 큰 포크가 메인 branch로 선택됩니다.
블록에 특정 컴퓨팅 성능이 포함된 경우에만 이를 인식하여 비가역화가 됩니다.
합의 알고리즘으로 dpos를 사용하는 EOS에서 블록의 인식 마크는 BP 서명이므로 BP 리스트는 EOSIO에서 중요한 역할을 합니다.
따라서 IBC의 라이트 클라이언트도 BP 리스트 교체를 유지해야하며 BP 리스트 교체에 대한 논리를 철저하게 분석해야합니다.  

먼저, 시스템 컨트랙트인 `eosio. system`의 `onblock ()` 액션에서 시스템은 매 분마다 
`update_elected_producers(timestamp)`를 통해 BP 리스트를 업데이트하려고 시도합니다.
이 액션은 일련의 검사 후 wasm 인터페이스인 `set_proposed_producers()`를 호출합니다. 이는 새로운 scheme과 현재 블록 번호를 `global_property_object`에 저장합니다.

두 번째는, 블록을 돌이킬 수 없게되면 현재 pending된 블록에 새로운 `header.new_producers` 목록이 설정되고 `global_property_object` 오브젝트가 리셋됩니다.
pending 중인 블록 번호는 `pending_schedule_lib_num`에 기록되며, 이 때 nodeos 로그에 새 목록이 표시됩니다.
자세한 로직은 `controller:: start_block() // Promote proposed schedule to pending schedule.`을 참조하기 바랍니다.
즉, `proposed schedule`에서 `pending schedule`로 새 목록을 변경하는데 약 325-336 블록이 필요합니다.
여기서부터 `block_header_state` 블록의 `pending_schedule.version`은 `active_schedule.version` 보다 1 만큼 더 커지게 됩니다.

세 번째는, `pending_schedule_lib_num`이 비가역화 되면 `active_schedule`이 `pending schedule`로 대체되고 전체 BP 교체 프로세스가 완료됩니다.
`pending schedule`에서 `active_schedule`까지 약 325-336 블록이 필요합니다.

IBC 시스템의 라이트 클라이언트는 신뢰할 수 있는 라이트 클라이언트를 실현하기 위해 이러한 forkdb 논리를 상속해야합니다.
그러나 라이트 클라이언트는 스마트 컨트랙트에서 구현되며 스마트 컨트랙트의 특성과 한계를 완전히 고려해야하므로 구현에 있어서 세부 사항에 많은 조정이 필요합니다.

**3.3.2 eosio::table("chaindb"), ibc.chain 스마트 컨트랙트의 "forkdb"**  

*1. Lightweight Client(lwc)의 LIB를 판별하는 방법*  
두가지 방식이 있습니다.
하나는 forkdb의 논리에 따라 전체 `confirm_count`, `confirmations` 및 기타 `block_header_state` 관련 정보를 유지하려면 블록이 추가 될 때마다 LIB를 계산하는 것입니다. 이 방법은 실시간 LIB 값을 정확하게 얻을 수 있다는 장점이 있습니다. 그러나 라이트 클라이언트의 경우 실시간 LIB 값이 아니라 블록을 돌이킬 수 없다는 점이 문제가 됩니다.
더 간단한 해결책이 있을까요?
위의 논리에 따르면, 3분의 2 이상의 BP가 활성화 되어있고 한 블록의 깊이가 336을 초과하면 해당 블록은 돌이킬 수 없어야하며 크로스 체인 트랜잭션을 확인하는데 사용될 수 있습니다. 이는 스마트 컨트랙트에서 forkdb의 복잡성을 단순화 할 수 있습니다.

ibc.chain 스마트 컨트랙트에서 `global` 테이블의 `lib_depth`는 깊이 값입니다.
연속 블록헤더에서 블록 깊이가 이 값을 초과하면 되돌릴 수 없는 것으로 간주되어 트랜잭션을 검증하는데 사용할 수 있습니다.
이 값은 336으로 설정해야합니다.
라이트 클라이언트가 2/3 이상의 BP가 활성 상태인지 확인하면 깊이가 336보다 큰 블록은 되돌릴 수 없는 것으로 간주됩니다.
그러나 스마트 컨트랙트에서 블록을 추가, 삭제하는 것은 CPU를 많이 사용합니다.
실제 테스트에 따르면 블록헤더를 추가하는데 0.5ms CPU와 블록헤더를 삭제하는데 0.2ms가 걸리므로 각 블록헤더에 0.7ms CPU가 필요합니다.
메인넷에 큰 변동이 있으면 LIB로 들어가지 않은 모든 블록을 롤백시킬 수 있을까요?
이것은 가능하며 실제로 일어났습니다.
따라서 크로스 체인 트랜잭션이 LIB에 들어간 후에 처리를 시작해야합니다.
현재 플러그인이 해당 역할을 수행할 수 있습니다.
BOSCORE 기술 팀에서 개발 한 ibc_plugin에서의 매개변수는 특정 깊이의 크로스 체인 트랜잭션만 처리하도록 설정되어 있습니다.
따라서 ibc_plugin의 깊이와 스마트 컨트랙트의 깊이는 336 이상을 합산합니다.
이것은 거대한 CPU 소비를 피하는 것을 목적으로 한 BOSIBC의 초기 설정입니다.
다음 업그레이드에서는 스마트 컨트랙트의 깊이를 336으로 설정하는 것이 고려 될 수 있으므로 릴레이의 깊이에 전혀 의존하지 않습니다. 
그러나 ibc_plugin 또는 ibc.chain의 깊이 값을 추가하면 크로스 체인 트랜잭션의 도착 시간이 직접 연장되어 사용자 경험에 영향을 미칩니다.
따라서, 양호한 사용자 경험을 잃지 않으면서 충분한 보안을 보장하기 위해 실제 상황에 따라 합리적인 가치를 결정하는 것이 필요합니다.

*2. table chaindb*  
Table chaindb는 ibc.chain 스마트 컨트랙트의 "forkdb" 입니다.
ibc.chain 스마트 컨트랙트에는 blog 구조가 없습니다.
eosio의 forkdb에서 사용되는 `block_header_state`와 달리 chaindb는 예약된 `Block_num, block_id, header, active_schedule, pending_schedule, blockroot_merkle, block_signing_key`라는 7개의 엘리먼트만을 많이 간소화 했습니다.
그 이유는 BP schedule은 많은 공간을 차지하기 때문입니다.
따라서 실제 schedule 정보를 다른 테이블 `prodsches`에 저장하고 chaindb의 ID 참조를 사용하여 메모리 및 wasm CPU 소비를 절약합니다.

*3. Lightweight Client의 제네시스 블록헤더*  
라이트 클라이언트에는 라이트 클라이언트의 제네시스 블록헤더와 같은 신뢰할 수 있는 시작점이 필요합니다.
제네시스 블록헤더 뒤의 모든 블록헤더에 대한 검증은 제네시스 블록헤더를 기반으로 진행됩니다.
블록 서명을 검증하려면 제네시스 블록헤더에 `block_header_state`의 `blockroot_merkle`, `active_schedule` 및 `header` 정보가 필요합니다.
소스 코드는 `chain::chaininit()` 입니다.
가장 중요한 제한사항은 제네시스 블록헤더의 `pending_schedule`이 `active_schedule`과 동일해야한다는 것입니다.
만약 제네시스 블록헤더의 `pending_schedule`과 `active_schedule`이 다르다면 이 블록은 BP schedule 교체 프로세스의 블록임을 의미하기 때문입니다.
BP schedule 교체 프로세스의 블록을 제네시스 블록헤더로 사용하는 경우 `active_schedule`이 `pending_schedule`로 대체 될 때까지 후속 블록을 동기화해야합니다.
이는 복잡성을 증가시킵니다.
따라서 이러한 블록헤더는 제네시스 블록헤더에 적합하지 않습니다.

*4. Lightweight Client가 새로운 블록헤더를 추가하는 방법*  
라이트 클라이언트가 헤더를 추가하는 방법은 eosio의 forkdb와 매우 유사합니다.
소스 코드는 ibc.chain의 `pushheader()`를 참조하기 바랍니다.
첫 번째 단계는 블록 번호로 최신 section에 연결할 수 있는지 확인하는 것 입니다.
두 번째 부분은 fork를 처리하고 오래된 branch를 삭제해야할 필요가 있는가에 대한 여부입니다.
ibc.chain에서는 여러 branch가 동시에 저장되지 않습니다.
대신, 포크 처리를 달성하기 위해 이전 branch가 최신 branch로 대체됩니다.
세 번째는 블록 ID로 최신 section에 연결할 수 있는지 확인해야 합니다.
네 번째 부분은 `block_header_state`를 구성하고 BP 서명을 확인합니다.
핵심 로직은 `block_header_state`를 구성하는 것인데, `BP schedule` 교체는 2/3 이상의 활성화된 BP가 생성한 블록임을 보장하기 위해 처리됩니다.(`section_type::add()` 참조)

*5. Section*  
Section은 chaindb의 핵심 개념과 혁신으로 연속된 블록헤더의 일괄처리를 의미합니다.
Section 관리는 ibc.chain의 핵심 로직입니다.
Section을 사용하는 목적은 CPU 소비를 줄이고 BP schedule 변경 또는 블록체인간 트랜잭션이 발생할 때만 일괄처리될 블록헤더들을 동기화하는 것입니다.
BP schedule 교체 프로세스에서는 어떤 section의 시작 블록헤더도 블록이 될 수 없습니다.
즉, 모든 section의 시작 블록헤더의 `pending_schedule.version`은 반드시 `active_schedule.version`과 같아야합니다.
그리고 section의 시작 블록헤더의 `active_schedule`은 반드시 이전 section의 `active_schedule`과 같아야합니다.
이렇게하면 두 section간에 BP schedule을 교체 할 필요가 없으며 각 BP schedule 교체의 전체 프로세스는 section **안에서** 완료되어야합니다.
이는 section 데이터의 신뢰성을 보장합니다.


#### 4. Inter-blockchain Transactions
*1. Inter-blockchain Transaction Trilogy*  
Inter-blockchain 트랜잭션은 세 가지 프로세스로 나뉩니다.
다음은 EOS 메인넷에서 BOS 메인넷으로 EOS 토큰을 전송하는 예입니다.
먼저, 사용자는 EOS 메인넷에서 Inter-blockchain 트랜잭션인 `orig_trx`를 시작하고 트랜잭션 정보는 EOS 메인넷 쪽의 `ibc.token` 스마트 컨트랙트의 `origtrx` 테이블에 기록됩니다.
트랜잭션이 LIB에 들어가면 EOS 메인넷 쪽의 IBC 릴레이(relay_eos)는 트랜잭션 및 트랜잭션 관련 정보(블록정보 및 머클패스)를 가져온 다음 BOS 메인넷 쪽의 릴레이(relay_bos)로 전달합니다.
relay_bos는 'cash' 트랜잭션을 구성하고 BOS 메인넷 쪽에서 `ibc.token` 스마트 컨트랙트의 **cash** 액션을 호출합니다.
호출이 성공하면 **cash** 함수에서 해당 토큰이 타켓 유저에게 발행됩니다.
**cash** 트랜잭션이 포함된 블록이 LIB에 들어오면, relay_bos는 `cash_trx` 및 관련 정보(블록정보 및 머클패스)를 가져온 다음 relay_eos에 전달하면 relay_eos는 **cashconfirm** 트랜잭션을 구성하고 EOS 메인넷 쪽에서 `ibc.token`의 **cashconfirm** 액션을 호출합니다. 이때, `ibc.token` 스마트 컨트랙트의 `orig_trx` 기록은 삭제합니다.
이로써 전체 블록체인 트랜잭션이 완료됩니다.

*2. 실패한 Inter-blockchain Transactions*  
Inter-blockchain 트랜잭션은 실패 할 수 있습니다.
예를 들어, 지정된 계정이 피어 체인에 존재하지 않거나 네트워크 환경이 좋지 않아 cash interface에 대한 호출이 실패할 수 있습니다.
이 때, 실패한 Inter-blockchain 트랜잭션은 롤백됩니다.
즉, 사용자 자산이 원래 상태로 롤백 되었음을 의미합니다.
그러나 현재 IBC 시스템은 Transaction-driven이며 실패한 IBC 트랜잭션은 롤백이 될 수 있기 전에 성공적으로 IBC 트랜잭션이 완료가 될때까지 기다려야한다.(참고: 후속 업그레이드는 가능한 빨리 실패한 트랜잭션을 롤백합니다)


*3. double-flower attacks와 같은 replay attacks를 방지하는 방법*  
double-flower attacks 방지는 두 단계로 나눌 수 있습니다:
1. 성공적인 크로스 체인 트랜잭션은 **cash** 를 한 번만 실행할 수 있습니다. 만약 그렇지 않으면 반복되는 cash가 발생하게 됩니다.
2. 각 **cash** 트랜잭션에 대하여 스마트 컨트랙트에 기록된 오리지날 트랜잭션 정보를 제거하기 위해 **cashconfirm** 을 실행해야합니다. **cashconfirm** 을 실행하기 위해 관련 정보를 원래 체인으로 다시 전송해야합니다. 그렇지 않으면 토큰이 대상 체인의 사용자에게 발행되면서 원래 체인의 토큰을 보내고자 하는 유저에게 반환됩니다. 즉, 가치가 복제되게 됩니다.

**cash** 함수는 ibc.token의 핵심 로직입니다.
`ibc.token` 스마트 컨트랙트는 최근에 cash 액션을 실행한 오리지날 트랜잭션 ID의 `orig_trx_id`를 기록하고 새 cash의 `orig_trx`의 블록 번호는 모든 `orig_trx`의 블록 번호보다 크거나 같아야합니다.
즉, cash는 반드시 오리지날 트랜잭션에 따라 오리지날 체인의 블록 순서대로 처리 되어야합니다.(cash가 실행될 때, 오리지날 체인에서 하나의 동일한 블록에 있는 크로스 체인 트랜잭션 순서는 임의로 순서를 정할 수 있습니다)
trx_id 점검과 결합하여 크로스 체인 트랜잭션이 한 번만 cash 액션을 실행할 수 있도록 보장 할 수 있습니다.

마찬가지로, cashconfirm 액션은 cash 트랜잭션 번호 즉, 'seq_num'를 점검합니다. 
'seq_num'은 반드시 오리지날 체인의 오리지날 트랜잭션 기록에서 대상 체인의 모든 cash 트랜잭션이 확실하게 제거 될때마다 하나씩 증가되어야 합니다.
이를 통해 double spend가 발생하지 않습니다.


### 5. ibc_plugin 
ibc_plugin의 역할은 두 부분으로 나뉩니다.
1. 라이트 클라이언트 동기화
2. Inter-blockchain 트랜잭션들의 전송
핵심 로직에 대해서는 `ibc_plugin_impl::ibc_core_checker()`를 참고할 수 있습니다.
ibc_plugin은 주로 net_plugin의 프레임워크를 나타냅니다.


### 6. Q&A
*1. 질문: 릴레이의 권한은 IBC 스마트 컨트랙트의 많은 액션에 사용되므로 IBC 시스템은 릴레이의 신뢰에 의존합니까?*   
답변: 현재 보안 및 fast function iteration을 위해 릴레이 권한이 추가되었습니다. IBC의 점진적인 개선 및 성숙과 함께 BOS IBC는 single-point risk를 피하기 위해 multiple relay 채널을 지원할 것입니다.

릴레이 권한을 검증할 때 고려해야 할 두 가지 사항이 있습니다.
1. ibc.chain 스마트 컨트랙트는 **section** 메커니즘을 사용하며 현재 로직은 오래된 section에 대한 블록을 추가할 수 없으며 section 앞에 블록헤더를 추가할 수 없습니다.
푸시해야할 블록의 범위가 1000-1300이라고 가정하고 누군가가 "pushsection" 액션을 호출 할 수 있다면 의도적인 공격자는 1100-1300을 먼저 푸시 할 수 있습니다. 결과적으로 1000-1100을 더 이상 푸시 할 수 없으므로 일부 크로스 체인 트랜잭션이 성공하지 못할 수 있습니다. (이 문제는 향후 버전의 최적화에서 고려 될 것입니다.)
2. IBC 시스템이 다수의 사용자 자산을 보유하고 있으며 시스템이 long-term market test를 통과하지 못했기 때문에 보안 위험을 줄이기 위해 릴레이 권한이 증가한다는 점을 고려해야합니다.


### 7. 업그레이드 계획
*1. PBFT consistency 알고리즘과 호환*  

*2. 보다 우아한 방식으로 여러 사이드 체인간의 Inter-blockchain 트랜잭션 지원*  

*3. "Token"보다 더 많은 유형의 데이터에 대한 IBC 지원*  


### 8. Reference
[Inter-blockchain Communication via Merkle Proofs with EOS.IO](https://steemit.com/eos/@dan/inter-blockchain-communication-via-merkle-proofs-with-eos-io) *Bytemaster*    
[Bitcoin](https://bitcoin.org/bitcoin.pdf) *Satoshi Nakamoto*   
[Chain Interoperability](https://static1.squarespace.com/static/55f73743e4b051cfcc0b02cf/t/5886800ecd0f68de303349b1/1485209617040/Chain+Interoperability.pdf) *Vitalik Buterin*   




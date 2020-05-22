
# soft_dwcas


A software implementation of the dwcas (`CMPXCHG16B` on x86) instruction



Algorithms built around CAS typically read some key memory location and remember the old value. Based on that old value, they compute some new value. Then they try to swap in the new value using CAS, where the comparison checks for the location still being equal to the old value. If CAS indicates that the attempt has failed, it has to be repeated from the beginning: the location is re-read, a new value is re-computed and the CAS is tried again. Instead of immediately retrying after a CAS operation fails, researchers have found that total system performance can be improved in multiprocessor systems—where many threads constantly update some particular shared variable if threads that see their CAS fail use exponential backoff, in other words, wait a little before retrying the CAS.



### soft_dwcas


The actual (hardware-)implementation is surpising and confusing (hence `soft_dwcas`, exposing its operation). It is (slightly) usefull in developing as mis-using the `CMPXCHG16B` instruction results in an instant crash, while the `soft_dwcas` let's one inspect results more easily (print-debugging).


    namespace lockless {

    template<typename MutexType>
    [[nodiscard]] HEDLEY_ALWAYS_INLINE bool soft_dwcas ( uint128_t * dest_, uint128_t ex_new_, uint128_t * cr_old_ ) noexcept {

        alignas ( 64 ) static MutexType cas_mutex;

        std::scoped_lock lock ( cas_mutex );

        bool check = not equal_m128 ( dest_, cr_old_ );

        std::memcpy ( cr_old_, dest_, 16 );

        if ( check )
            return false;

        std::memcpy ( dest_, &ex_new_, 16 );
        return true;
    }

    } // namespace lockless



Implementing this little function made me understand the difference between lockless and concurrent (mutex based) programming. I've listed in random order some of my thoughts:

* mutexes prevent concurrency
* `soft_dwcas` implies full serialization (i.e. has no real-world use other than real-world developing)
* `soft_dwcas` cannot be used to test concurrency itself, just that the mechanics [of lockless progression] work
* the mechanism used in lockless-programming is locks
* `dwcas` uses 'the data' as a mutex, *intrusive* mutexes
* as many intrusive mutexes as there are data points (as compared to 1, or a few)
* `CMPXCHG24B` (fully lockless dl-list) and `CMPXCHG32B`?
* illustrative, a mental model (`CMPXCHG16B` is 'similar' to `soft_dwcas` minus the mutex)
* below a lockless implementation (for completeness sake)



### dwcas (win/nix/ios)


    namespace lockless {

    [[nodiscard]] HEDLEY_ALWAYS_INLINE bool dwcas ( volatile uint128_t * dest_, uint128_t ex_new_, uint128_t * cr_old_ ) noexcept {
    #if ( defined( __clang__ ) or defined( __GNUC__ ) )
        bool value;
        __asm__ __volatile__( "lock cmpxchg16b %1\n\t"
                              "setz %0"
                              : "=q"( value ), "+m"( dest_ ), "+d"( cr_old_->hi ), "+a"( cr_old_->lo )
                              : "c"( ex_new_.hi ), "b"( ex_new_.lo )
                              : "cc" );
        return value;
    #else
        return _InterlockedCompareExchange128 ( ( volatile long long * ) dest_->lo, ex_new_.hi, ex_new_.lo,
                                                ( long long * ) cr_old_->lo );
    #endif
    }

    } // namespace lockless

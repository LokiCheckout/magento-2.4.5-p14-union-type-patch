# PHP 8.1 union type + mixed patches for Magento 2.4.5-14

**Patch to fix DI compilation errors due to PHP 8.1+ union types and the `mixed` keyword, used in Magento 2.4.5-14.**

## Backgrounds
PHP 8.1 supports various new features and Magento 2.4.5 was meant to support them all properly. Unfortunately, issues arise when using PHP 8.1+ union types and the `mixed` keyword. Specifically, the following errors are given when running `setup:di:compile`:

```bash
Type "mixed" cannot be nullable
```

This `mixed` error is fixed by an official quality patch ACSD-55031.

```bash
Call to undefined method ReflectionUnionType::getName()
```

The `ReflectionUnionType` error is fixed with the patch in this repository. For this error, a patch is provided in this repository.

## Requirements
- Magento 2.4.5-p15 only (issues are fixed in Magento 2.4.6)
- cweagans/composer-patches 1.0

## WARNING: Incompatible with Magento Quality Patch
The official Magento Quality Patch ACSD-55031 fixes the issue of the `mixed` error. Unfortunately, because Quality Patches are not delivered as composer patches and because both
fixes require modification of the same file, our patch approach is **not compatible** with the regular ACSD-55031 patch.

Instead, the ACSD-55031 patch is offered here as a composer patch instead.

Other Quality Patches (that do not modify the files of these patches) should work fine.

## Usage
Copy the files `union-types-patch.diff` and `ACSD-55031_composer-patch.diff` to a folder `patches/` in your Magento project.

Run the following commands to install `cweagans/composer-patches`:
```bash
composer require cweagans/composer-patches:^1
```

Open up your Magento `composer.json` file and add the following:
```json
{
  "extra": {
    "patches": {
      "magento/framework": {
        "ACSD-55031 quality patch": "patches/ACSD-55031_composer-patch.diff",
        "Support union types": "patches/union-types-patch.diff"
      }
    }
  }
}
```

Re-run composer installation (`composer update` will do) to apply these composer patches.

## End result
After applying both patches (using either scenario), the file `vendor/magento/framework/Interception/Code/Generator/Interceptor.php` should look like the following (with irrelevant parts left out):

```php
class Interceptor extends EntityAbstract
{
    /**
     * Get return type
     *
     * @param \ReflectionMethod $method
     * @return string|null
     */
    private function getReturnTypeValue(\ReflectionMethod $method): ?string
    {
        $returnTypeValue = null;
        $returnType = $method->getReturnType();
        if ($returnType) {
            if ($returnType instanceof \ReflectionUnionType || $returnType instanceof \ReflectionIntersectionType) {
                return $this->getReturnTypeValues($returnType, $method);
            }

            $className = $method->getDeclaringClass()->getName();
            $returnTypeValue = ($returnType->allowsNull() && $returnType->getName() !== 'mixed' ? '?' : '');
            $returnTypeValue .= ($returnType->getName() === 'self')
                ? $className ? '\\' . ltrim($className, '\\') : ''
                : $returnType->getName();
        }

        return $returnTypeValue;
    }

    /**
     * Get return type values for Intersection|Union types
     *
     * @param \ReflectionIntersectionType|\ReflectionUnionType $returnType
     * @param \ReflectionMethod $method
     * @return string|null
     */
    private function getReturnTypeValues(
        \ReflectionIntersectionType|\ReflectionUnionType $returnType,
        \ReflectionMethod $method
    ): ?string {
        $returnTypeValue = [];
        foreach ($method->getReturnType()->getTypes() as $type) {
            $returnTypeValue[] =  $type->getName();
        }

        return implode(
            $returnType instanceof \ReflectionUnionType ? '|' : '&',
            $returnTypeValue
        );
    }
}
```
